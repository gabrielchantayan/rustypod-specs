<!-- Trusted provenance: F-f41ad06d9af3; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-f41ad06d9af3

## Purpose

F-f41ad06d9af3 performs an ordered sequence of indirect calls, derives values and addresses using fixed-width arithmetic, modifies memory through computed addresses and named fixed-address storage, and ends with an unresolved four-argument indirect transfer.

No application-level purpose is established. Constants resembling printable characters do not establish text output, serialization, formatting, or a device protocol.

This contract distinguishes:

- **Established static evidence:** explicit value conversions, typed pointer accesses, captured-value dependencies, and represented ordering.
- **Inferred intent:** possible interpretations that are not acceptance requirements. No application-level interpretation is adopted.
- **documented_only:** behavior that the supplied evidence does not resolve, or that this contract does not fully enumerate.

All requirements are conditional on the supplied static representation accurately reflecting execution. No runtime tests have been performed.

## Interface

### Explicit inputs

| Input | Represented width | Established use |
|---|---:|---|
| First argument | 32 bits | No explicit consumption in the body. |
| Second argument | 32 bits | No explicit consumption in the body. |
| Third argument | Unsigned 32 bits | Bits 10–19 provide an early call argument. |

The extracted ten-bit field ranges from 0 through 1023. Shifting it left by 11 produces multiples of 2048 from 0 through 0x1FF800. One preliminary call receives that result in two argument positions.

The absence of explicit uses of the first two arguments does not exclude implicit calling-convention dependencies.

### Implicit inputs and preconditions

Execution depends on entry control-flow state, embedded executable references, indirect-call results, incompletely attributed call-associated values, named global storage, and memory reached through computed or loaded addresses.

Every reached call destination and memory location must support the actual operation, including its required permissions and alignment. The represented body provides no bounds checks, null checks, address validation, alignment repair, or allocation-size checks.

No buffer extent, object layout, ownership transfer, or memory lifetime follows from the interface.

### Outputs and completion

There is no explicit scalar return value. Observable outputs include dependency activity, memory changes, and the terminal indirect transfer.

The terminal transfer receives four represented arguments:

1. 0x72.
2. A captured 32-bit value from final-record base + 0x64.
3. A previously captured 32-bit value from Double-link base + 0x14.
4. A later captured 32-bit value from final-record base + 0x74.

These are retained values, not instructions to reread their originating locations at transfer time.

The static representation includes a return after the terminal invocation. Whether that invocation returns, and whether control actually returns to the original caller, remain **documented_only**.

## Behavior

### Access-width boundary

**Explicitly typed pointer accesses and named-global references have different evidentiary strength.**

Computed-address pointer accesses explicitly establish 8-, 16-, or 32-bit access widths in the supplied representation. An explicitly unsigned 8-bit pointer load produces a value from 0 through 255; widening that value to unsigned 32 bits is zero-extension. A one-byte destination establishes an 8-bit store.

The declarations of named fixed-address globals are absent. Their names, address suffixes, and overlapping-symbol warning do not establish storage width or signedness.

Consequently:

- Converting a global’s value to unsigned 32 bits establishes the converted value’s width, not the load width or zero-extension.
- Capturing a global’s value in a 32-bit holder establishes the retained representation, not a 32-bit memory read.
- Narrowing an expression to one byte before assigning it to a global establishes an intermediate value width, not the global’s store width.
- Assigning an integer constant to a global does not establish its storage width.
- Copying one named global to another does not establish either access width or either declaration’s signedness.

The storage widths, signedness, conversion details, and affected byte ranges of these named-global accesses are **documented_only** pending declarations or instruction-level evidence.

### Preliminary dependency activity

A long straight-line sequence of indirect calls precedes the first explicit pointer-based memory write. Represented argument counts range from zero through four.

Arguments include constants, shifted results, low and high components of paired results, and additional call-associated values whose producers are incompletely established. Some results are represented as 64-bit quantities, but this alone does not establish an application-level 64-bit return type.

Calls whose results are unused remain significant. Their effects on memory, globals, ownership, failures, and control flow are unknown. They cannot be removed merely because no explicit return value is consumed.

No explicit loop or conditional branch appears in the represented body. Reaching later effects nevertheless requires preceding dependencies to return and preceding accesses to succeed.

### Arithmetic

The dominant scalar width is 32 bits. Established unsigned operations include masking, logical right shifts, and width-limited left shifts. Masking must precede shifting where represented: retaining 24 low bits and then shifting left by six yields at most 0x3FFFFFC0.

Unsigned 64-bit right shifts by 34, 36, and 44 correspond to logical shifts of the high 32-bit component by two, four, and twelve. They are not 32-bit shifts by the original counts.

Several retained values are explicitly zero and remain zero under subsequent shifts. This establishes those local argument values, not preservation of global storage across calls.

Signed left shifts, signed multiplication, overflow, and signed-word-to-address conversion require target-specific interpretation. Their exceptional and overflowing cases remain **documented_only**.

A late signed right shift uses the low eight bits of Working base as its count. No guard restricts that count below 32. Oversized counts and negative operands require target-specific interpretation.

### Retained-value terminology

These labels denote captured values, not inferred object fields.

| Label | Meaning |
|---|---|
| **Small scale** | Unsigned result obtained by masking a shifted call result with 0x7FFF and shifting right by six. |
| **Double-scale base** | Twice Small scale. |
| **Seed value** | The unsigned 32-bit conversion of the named global associated with 0x05. |
| **Anchor word** | Explicit 32-bit pointer load from Seed value + 0x54, captured before that location is overwritten. |
| **Adjusted base** | Four times Anchor word, minus 0x96. |
| **Working base** | Four times Adjusted base, retained as unsigned 32 bits. |
| **Link word** | A later explicit 32-bit load at offset 0x74 from a zero-extended, explicitly loaded unsigned byte. |
| **Double-link base** | Twice Link word. |
| **Base slot** | Named global associated with 0x087FBBBC. |
| **Shared base** | Base slot’s value converted into the retained signed 32-bit base. |
| **Late fixed capture** | Value from the named global associated with 0xA9, retained in 32 bits. |

Under defined interpretation of its preceding signed shift, Small scale is a multiple of four from 0 through 508. Seed value is not established as byte-sized or restricted to 0–255.

Arithmetic descriptions retain the target qualifications above. Unsigned 32-bit address expressions retain modulo-2³² arithmetic; signed-address expressions are not silently assigned equivalent wraparound semantics.

### Opening pointer-based memory effects

The final preliminary call supplies a paired result. Its high component supplies one base; its low component scaled by 256 supplies another.

The opening accesses preserve this order:

- A 32-bit store at high-component base + 0x14.
- An 8-bit store at an earlier retained base + 9.
- A 32-bit store at scaled-low-component base + 0x54.
- A second 8-bit store to the earlier offset-9 destination.
- A 32-bit clear at another derived base + 0x14.
- An explicitly unsigned 8-bit read at that base + 5.
- A 32-bit clear at Double-scale base + 0x14.
- A 32-bit store of the zero-extended captured byte at the earlier base + 0x34.

The earlier cleared offset-0x14 location is subsequently reloaded as 32 bits. That reload cannot unconditionally be replaced with zero: intervening destinations may alias, and ordinary private-memory behavior is not established.

### Early named-global accesses and captures

After another clear at Double-scale base + 0x14, two consecutive byte stores occur at Double-scale base + 0x15. Only then is the global associated with 0x05 evaluated and converted to unsigned 32 bits as Seed value.

Next, a one-byte-narrowed value derived from Double-scale base is assigned to the global associated with 0x01. Its actual store width is unresolved.

Seed value supplies the base for:

- An explicit byte store at Seed value + 0x15, using the low byte of Small scale.
- A later explicit 32-bit read at Seed value + 0x54, producing Anchor word.
- A subsequent explicit 32-bit overwrite at that same offset, using the earlier reload.

Intervening derived writes occur before Anchor word is captured. Anchor word nevertheless remains the pre-overwrite capture. Later changes to the global at 0x05 do not recapture Seed value.

A derived 32-bit store at Double-scale base + 0x14 precedes assignment of Anchor word shifted left by one to the global associated with 0x54. The supplied value is a represented signed 32-bit expression; the destination width is unknown.

Afterward, Double-scale base + 0x14 receives Anchor word twice. Two consecutive byte stores at Double-scale base + 0x15 supply the low byte of four times Anchor word. A one-byte-narrowed Double-scale-base value is then assigned to the global associated with 0x15.

Neither named-global assignment establishes a four-byte range at 0x54 or a one-byte range at 0x15.

### Derived-address construction

Further accesses use loaded words and explicitly zero-extended unsigned bytes as address bases. Repeated offsets do not establish a structure type.

Before Adjusted base is formed, the represented effects include:

- Repeated 32-bit self-value stores at Anchor word + 0x24.
- Repeated byte stores at derived offset-9 and offset-0x15 destinations.
- Word-based links through offsets 4, 0x44, 0x54, 0x64, and 0x74.
- Two separate unsigned-byte reads through a loaded base + 5, with intervening writes; their results cannot automatically be equated.
- Replacement of an earlier byte-derived link with the later captured Link word.

Link word then determines explicit stores of 0x3A at three destinations: an 8-bit store at four times Link word, a 16-bit store at eight times Link word, and a 32-bit store at four times Link word + 0x20.

Working base + 0x20 next receives successive 32-bit stores of 0x20, 0x2F, the zero-extended byte explicitly read from Working base + 4, and 0x20.

The routine captures Shared base from Base slot, then assigns 0x20 to the named global associated with 0x60. Neither named-global reference has an established memory-access width. Shared base remains captured even if subsequent writes alter Base slot.

### Repeated region

A finite repeated region follows the assignment at 0x60. Each represented group preserves four ordered pointer-based effects:

1. A 32-bit store of four times Link word at a scaled base + 0x54.
2. A byte store at Shared base + 0x19, supplying the low byte of Working base.
3. A byte store at Working base + 0x11, supplying the low byte of Shared base.
4. A 32-bit store of the scaled base at Shared base + 0x34.

The first group uses eight times Anchor word as the scaled base. Later groups use eight times Adjusted base.

The exact repetition count is **documented_only** in this contract. This is an enumeration limit, not evidence that repetition is optional or dynamically unbounded. Full access-trace equivalence requires preserving the supplied multiplicity.

### Late named-global activity and fresh observations

After the repeated region, Late fixed capture is obtained from the global associated with 0xA9. A separate explicit 32-bit value is then captured from four times Link word + 0x74.

Next, the global associated with 0x3A is assigned an expression formed by first converting Adjusted base to a character-sized value, multiplying by eight, and adding 0x30. Character signedness and the destination declaration are absent. This does not establish an 8-bit store or unconditional low-eight-bit result.

The routine then captures:

- A 32-bit value from Double-link base + 0x14 for the terminal third argument.
- A 32-bit value from Working base + 4 for later storage at Shared base + 0x74.

Derived byte writes follow at Double-link base + 0x0D and Working base + 8, +0x0D, and +9. Double-link base + 0x54 receives Double-link base as 32 bits.

The global associated with 0x67 is then assigned a one-byte-narrowed Late fixed capture. This does not establish the global’s store width. Late fixed capture itself is not reread from 0xA9.

After a 32-bit read from Double-link base + 0x44, the global associated with 0x67 is evaluated and assigned to the global associated with 0x21. Neither access width nor the destination conversion is established.

Two later evaluations of the global associated with 0x67 each supply a separate explicit byte store at Working base + 9:

- Before the first, intervening effects include a byte store at Working base + 0x10; two word stores at Shared base + 0x54 separated by a byte store at Double-link base + 1; a word read at Working base + 0x40; and word stores at Shared base + 0x74, Working base + 0x10, and Shared base + 0x44.
- Before the second, the routine writes a byte at Double-link base + 9, stores Late fixed capture as 32 bits at Working base + 0x44, and loads a new destination base from Working base + 0x54.

These are fresh global evaluations, not reuse of the initial narrowed value. Their source-load widths and signedness remain unresolved; their destinations are explicitly one byte wide.

### Final construction and transfer

The new destination base receives byte updates and a 32-bit value of decimal 100 at offset 0x54. Working base + 9 and +0x44 are overwritten using the captured destination base.

A 32-bit load from Double-link base + 4 supplies final-record base. A 16-bit store of 0x65 occurs at twice that base.

The terminal second argument is captured from final-record base + 0x64. Before the fourth argument is captured from final-record base + 0x74, additional byte and word writes occur through the new destination base.

A further 32-bit destination base is loaded from Double-link base + 0x64. It receives the captured second argument at offset 0x54. A final 32-bit store at twice the captured second argument supplies that further destination base. The terminal transfer follows.

## Side effects

Established pointer-based effects include 8-, 16-, and 32-bit stores, repeated overwrites, explicitly unsigned byte loads, and reads through loaded or call-derived addresses.

Named-global reads and assignments are also established, but their physical widths, signedness, and byte ranges are **documented_only**.

No disjointness is established. Explicit 32-bit accesses at offset 0x14 overlap explicit byte accesses at offset 0x15 when they use the same base; offsets 0x10 and 0x11 have the corresponding relationship. Numerical overlap results depend on endianness.

For named globals, overlap analysis additionally requires their missing widths. No claim is made that an assignment associated with 0x60 covers exactly 0x60–0x63, or that the capture associated with 0xA9 reads exactly 0xA9–0xAC.

No explicit allocation or release is identified in the pointer-based phase. Unknown dependencies may allocate, release, transfer ownership, or invalidate memory.

There is no represented error status, rollback, recovery branch, or cleanup path. Faults or nonreturning dependencies can prevent later effects; earlier effects are not represented as undone.

Atomicity, synchronization, reentrancy, device-memory behavior, and interrupt safety are **documented_only**.

## Dependencies

| Opaque ID | Scope | Established relationship |
|---|---|---|
| F-unknown-001 | Initial indirect destination | Derived from implicit entry control-flow state; invoked first. |
| F-unknown-002 | Remaining preliminary call sites | Ordered unresolved calls whose results and call-associated values feed later behavior. |
| F-unknown-003 | Terminal indirect destination | Receives four represented arguments beginning with 0x72. |

These are documentation identities. F-unknown-002 groups call sites; it does not assert a common callee.

Calling conventions, global preservation, memory ownership, side effects, failure behavior, and return guarantees remain **documented_only**.

## Acceptance examples

These are conditional static deductions, not executed tests.

### Explicit field extraction

A third argument of 0x000003FF produces field zero. A value of 0x00000400 produces field one and shifted argument 0x800. A value of 0xFFFFFFFF produces field 1023 and shifted argument 0x1FF800.

### Seed conversion depends on the missing declaration

If the global associated with 0x05 is an unsigned 8-bit object containing 0xFE, Seed value is 0x000000FE.

If it is instead a signed 8-bit object whose representation denotes −2, conversion to unsigned 32 bits produces 0xFFFFFFFE.

Under those respective assumptions, unsigned addition of 0x54 produces 0x152 or 0x52. Therefore, neither the seed’s range nor its derived addresses may be specified solely from the byte pattern 0xFE.

### Explicit unsigned byte loads are different

If an explicitly unsigned byte-pointer read observes 0xFE, its widened unsigned 32-bit value is 0x000000FE. That conclusion does not depend on the missing named-global declarations.

### Narrowing does not establish global store width

If Late fixed capture is 0x123456AB, the one-byte intermediate supplied to the global at 0x67 has bit pattern 0xAB. Requiring exactly one changed byte at that global would need additional declaration or instruction evidence.

The later explicit 32-bit store of Late fixed capture at Working base + 0x44 supplies 0x123456AB regardless of subsequent changes to the original global at 0xA9.

### Fresh global evaluations remain significant

If a later evaluation of the global at 0x67, after intervening effects and conversion to the explicit byte destination, produces 0x42, the corresponding Working base + 9 store supplies 0x42. Reusing an earlier 0xAB would violate that dependency. The two later evaluations need not produce equal destination bytes.

### Captured bases and terminal values remain retained

If Base slot converts to Shared base 0x1000, later Shared-base-relative accesses use that capture even if Base slot subsequently changes.

Likewise, changing memory that originally supplied the terminal third argument does not replace its earlier capture. The terminal first argument remains 0x72.

## Uncertainties

The following remain **documented_only**:

- Application purpose and actual calling convention.
- Missing named-global declarations, including storage widths and signedness.
- Named-global extension, truncation, byte ranges, and overlap consequences.
- Character conversion and destination conversion for the assignment associated with 0x3A.
- Incompletely attributed call-associated values and unresolved dependency effects.
- Signed overflow, shifts, address conversion, and oversized shift counts.
- Endianness, alignment, permissions, memory classification, and object boundaries.
- Exact repeated-region multiplicity as an enumeration limit of this contract.
- Ownership, concurrency, recovery, and terminal return guarantees.

Unknown storage widths do not make the named-global evaluations and assignments optional. Their established ordering and capture-versus-reread relationships remain requirements.

## Evidence

The supplied static body supports the apparent three-word interface, ten-bit extraction, ordered indirect calls, explicit arithmetic conversions, width-specific pointer accesses, named-global references, repeated stores, and terminal four-argument invocation.

The fixed-global declarations and instruction-level access evidence are absent. Width claims above therefore attach to explicit pointer accesses or retained values, not automatically to named-global storage.

The overlapping-symbol warning does not resolve those missing declarations.

No tools, external implementation access, runtime execution, tests, memory traces, or independent instruction-level validation were available or performed.
