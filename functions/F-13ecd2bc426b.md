<!-- Trusted provenance: F-13ecd2bc426b; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-13ecd2bc426b

## Purpose

F-13ecd2bc426b describes a sequence of narrow memory stores, wider state stores, and calls to externally defined operations. Twelve call results supply encoded destination addresses and 16-bit values. Most stages also clear a reset-associated global location. The sequence ends with additional calls and an invalid-instruction boundary, not an established normal return.

This is a source-facing behavioral contract based only on the supplied static representation. Its validity is conditional on that representation accurately describing reachable execution. The supplied material explicitly reports invalid instruction data and overlapping global symbols. Those limitations prevent treating the sequence as a verified, executable implementation.

A hardware-control or initialization purpose is plausible from the repeated stores and reset-associated symbol, but is **inferred intent**, not established behavior. No device protocol, application-level operation, or successful completion condition is established.

## Interface

### Explicit and implicit inputs

The displayed interface has no explicit parameters and no declared return value. Nevertheless, the body depends on four incoming machine-state values:

| Contract label | Width and interpretation | Use |
|---|---|---|
| **P** | 32-bit pointer/address representation | Initial 16-bit store destination and source of address bits |
| **B** | 32-bit address-like value | Base for initial 32-bit stores at byte offset 20 |
| **W** | Unsigned 32-bit word | Stored data, shifted data, and later destination derivation |
| **L** | 32-bit address-like value | Determines the first indirect call target |

These labels are contract notation, not source identifiers or declared arguments. The actual entry convention that supplies them is unknown.

The routine additionally depends on:

- Twelve externally returned values represented as 64-bit quantities.
- Writable memory at all derived store destinations.
- A reset-associated global location whose address, size, and overlap relationships are not established.
- Executable targets for all described calls.
- External execution state used or modified by those targets.

### Width conventions

For this contract:

- A halfword store writes 16 bits.
- A word store writes 32 bits.
- **low32(R)** and **high32(R)** designate the low and high 32 bits of a represented 64-bit result.
- **low16(X)** designates the low 16 bits of a value.
- Unsigned shifts discard shifted-out bits and introduce zero bits.
- A shifted or multiplied 32-bit word retains only its low 32 bits where the representation uses unsigned 32-bit arithmetic.

The representation also performs address arithmetic through signed 32-bit types. Wraparound, overflow, pointer validity, and alignment behavior for those operations are not established. Accordingly, examples involving such arithmetic are conditional on the target machine supporting the stated effective address.

### Outputs and completion

There is no established returned value or normal-return path.

The observable outputs are:

1. Ordered memory-store attempts.
2. Ordered external calls and their supplied argument bit patterns.
3. Any effects of those calls.
4. Eventual arrival at an invalid-instruction boundary, if every preceding operation completes and returns as needed.

No explicit error code, success indication, allocation result, or ownership transfer is exposed.

## Behavior

### Ordering and execution conditions

The order below is the order shown by the supplied static representation. Every later operation is conditional on all earlier stores completing and all earlier calls returning.

Repeated stores to the same address are distinct operations in this contract. They must not be collapsed into a single store when assessing the displayed behavior. Whether the actual binary emits every represented store, and whether hardware observers see them in that order, remain unverified.

There are no explicit conditional branches, retries, validation steps, or data-dependent termination checks in the displayed body.

### Initial stores and first call

Before any external call, the routine performs these stores in order:

1. Store **low16(P)** to address **P**.
2. Store the same 16-bit value to **P** again.
3. Store the same 16-bit value to the effective address obtained by doubling the numeric address **P**.
4. Store the 32-bit word **W** to address **B + 20**.

The doubling in the third store is numeric address arithmetic, not an additional element offset within an established array.

The routine next invokes dependency **F-0001** at a target formed from **L + 0xFFE**. No explicit arguments are shown, and its result is not used by the displayed body. This does not establish a genuine zero-argument interface or absence of implicit inputs.

### Result decoding

Twelve result-producing calls occur in sequence. Number their represented 64-bit results **R₁** through **R₁₂**.

Each result is decoded using one of two formats.

**Format A**

- Destination address: **low32(R) >> 8**.
- Stored value: **low16(R >> 8)**.
- Two consecutive 16-bit stores write that value to that destination.

The value therefore contains result bits 8 through 23. The address contains low-result bits 8 through 31, moved down by eight positions. The high 32 bits do not affect these stores.

**Format B**

- Destination address: **(low32(R) AND 0x07FFFFFF) >> 3**.
- Stored value: **low16((low32(R) << 5) >> 8)**, with the left shift performed at 32-bit width.
- Two consecutive 16-bit stores write that value to that destination.

For the final 16-bit value, Format B selects low-result bits 3 through 18. Its destination selects low-result bits 3 through 26. The mask prevents bits 27 through 31 from contributing to the address.

Both formats produce numeric destinations no greater than **0x00FFFFFF**. Neither format ensures two-byte alignment or excludes address zero.

### Twelve-stage sequence

Each stage consists of a result-producing call, the indicated pair of decoded stores, the additional stores listed below, and a subsequent consumer call.

For compactness:

- **T = (W AND 0x0FFFFFFF) >> 18**.
- **D** is the low 32 bits of **W × 16**.
- **S** is the low 32 bits of **W << 8**.
- **G** denotes the reset-associated global location.
- “Clear G” means an assignment of zero whose storage width is not established.

| Stage | Result dependency | Format | Additional stores, after the decoded pair and before the consumer |
|---|---|---|---|
| 1 | F-0101 | A | Clear G; store W at B + 20 |
| 2 | F-0102 | A | Clear G; store W at B + 20 |
| 3 | F-0103 | A | Store low16(T) at T twice, then at 2 × T; store W at B + 20 |
| 4 | F-0104 | A | Clear G; store W at D + 20 |
| 5 | F-0105 | B | Clear G; store W at D + 20 |
| 6 | F-0106 | B | Clear G; store W at D + 20 |
| 7 | F-0107 | B | Clear G; store W at D + 20 |
| 8 | F-0108 | B | Clear G; store W at D + 20 |
| 9 | F-0109 | A | Clear G; store S at D + 20 |
| 10 | F-0110 | B | Clear G; store S at D + 20 |
| 11 | F-0111 | B | Clear G; store S at D + 20 |
| 12 | F-0112 | A | Clear G; no subsequent word store is shown |

Stage 3 does **not** explicitly clear G. Its three extra halfword stores may nevertheless overlap G or another destination; no disjointness assumption is justified.

The stage-3 derivation first retains 28 bits of W and ultimately shifts them right by 18. Consequently, **T ranges from 0 through 1023**. Both its halfword value and its first destination have the same numeric value. The doubled destination ranges from 0 through 2046.

### Consumer arguments

Each stage’s consumer is distinct for purposes of this contract. The second argument carries **high32(Rᵢ)** as a 32-bit bit pattern, represented through a signed integer type.

The following constants are explicit argument values, not asserted object addresses:

| Stage | Consumer dependency | Third-argument constant |
|---|---|---|
| 1 | F-0201 | 0x0890FF45 |
| 2 | F-0202 | 0x0890FFAD |
| 3 | F-0203 | 0x08910015 |
| 4 | F-0204 | 0x0891007D |
| 5 | F-0205 | 0x089100E5 |
| 6 | F-0206 | 0x0891014D |
| 7 | F-0207 | 0x089101B5 |
| 8 | F-0208 | 0x0891021D |
| 9 | F-0209 | 0x08910285 |
| 10 | F-0210 | 0x089102ED |
| 11 | F-0211 | 0x08910355 |

Argument behavior is:

- **Stage 1:** four explicit arguments: zero, high32(R₁), the listed constant, and the low 32 bits of W shifted left four places.
- **Stage 2:** three explicit arguments: zero, high32(R₂), and the listed constant.
- **Stage 3:** three explicit arguments: T used as a pointer-valued argument, high32(R₃), and the numeric sum of T and the listed constant.
- **Stages 4–11:** three explicit arguments: zero, the corresponding high32 result, and the listed constant.
- **Stage 12, F-0212:** four explicit arguments: **0xFFFFFF1C**, **high32(R₁₂) − 0x264**, **0x4B**, and zero.

The subtraction for stage 12 is shown through a signed 32-bit representation. Its overflow semantics are unknown. The first argument’s bit pattern is 0xFFFFFF1C; if interpreted as signed 32-bit, it is −228.

The displayed body does not use any consumer return value.

### Terminal sequence

After F-0212, the routine calls **F-0301**, **F-0302**, **F-0303**, and **F-0304**, in that order. No explicit arguments or used results are shown.

The supplied representation then terminates at an invalid-instruction marker. It does not establish a deliberate application-level halt, a recoverable exception, a processor reset, or a normal return.

## Side effects

### Direct memory effects

If the full displayed sequence is reached, its direct operations include:

- **30 halfword-store attempts:** three initial stores, 24 decoded-result stores, and three extra stage-3 stores.
- **12 word-store attempts:** one initial store and one in each of stages 1–11.
- **11 assignments of zero** to G, with unknown storage width.

These are counts of operations, not counts of unique addresses. Aliasing can cause multiple categories to affect the same bytes. Word and halfword stores may partially overlap.

The body shows no explicit dereference reads of the store destinations. Values are derived from incoming state and call results, rather than from the previous contents of those destinations. Actual memory-access behavior beneath the represented stores is unknown.

### External effects and ownership

There are **29 call sites** before the invalid-instruction boundary: one initial call, 12 result-producing calls, 12 consumer calls, and four terminal calls.

No assumptions are made that these dependencies:

- Preserve G, other globals, or previously written memory.
- Preserve implicit machine state.
- Return normally.
- Avoid allocation, release, I/O, or other external effects.
- Treat pointer-valued arguments as borrowed rather than owned.

The displayed body itself contains no recognizable allocation or release operation. This does not establish that the complete execution allocates or releases nothing.

No ownership of P, B-derived storage, decoded destinations, or T-derived storage is established. Valid writable storage is an execution precondition, not an ownership guarantee.

## Dependencies

The following are document-local opaque function IDs. They distinguish call sites without claiming recovered external identities or equivalence between targets.

| IDs | Established role |
|---|---|
| F-0001 | Initial indirect call, target derived from incoming L |
| F-0101 through F-0112 | Ordered calls whose results are represented as 64-bit quantities |
| F-0201 through F-0212 | Ordered consumers of explicit stage arguments |
| F-0301 through F-0304 | Final four calls before the invalid-instruction boundary |

For F-0101 through F-0112, the representation supplies fixed numeric target expressions. For F-0201 through F-0211, the displayed target is the corresponding table constant plus **0xFFE**. F-0212 instead uses **0x089103BD + 0x7F6**. F-0301 uses **0x089104E3 + 0x7F6**.

These expressions establish apparent target selection, not a validated instruction-set mode, ABI, callable function boundary, or executable mapping. Dependency signatures and behavioral contracts are **documented_only**.

## Acceptance examples

These examples are conditional static expectations, not executed tests.

### Initial store order

Given P = **0x00001234**, B = **0x00002000**, and W = **0x89ABCDEF**, with all required destinations writable:

- The first two halfword stores write **0x1234** to **0x1234**.
- The third writes **0x1234** to **0x2468**.
- The next word store writes **0x89ABCDEF** to **0x2014**.
- F-0001 follows these stores.

This example makes no assertion about byte order within each store.

### Format A decoding

For a Format A result whose low 32 bits are **0x12345678**:

- Both halfword stores target **0x00123456**.
- Both store **0x3456**.
- Changing only the result’s high 32 bits leaves these two stores unchanged, but changes the second explicit argument to the stage’s consumer.

### Format B decoding

For a Format B result whose low 32 bits are **0x12345678**:

- Masking and shifting produces destination **0x00468ACF**.
- The 16-bit value is **0x8ACF**.
- The destination is odd. The displayed body does not realign or reject it.

Execution of these stores is conditional on the target’s alignment rules and memory mapping.

### Stage-3 low-address behavior

For W = **0x0FFFFFFF**, T is **0x3FF**. Stage 3 attempts two halfword stores of **0x03FF** to address **0x3FF**, followed by one to **0x7FE**.

Its consumer receives T as its first explicit argument. No explicit G clear occurs in this stage.

For W = zero, T and all three extra-store destinations are zero. There is no displayed null-address guard.

### Shift truncation

For W = **0xF1234567**:

- D is **0x12345670**.
- S is **0x23456700**.
- Stages 4–8 write W at D + 20.
- Stages 9–11 write S at that same derived destination.

These expectations require the subsequent address addition to have the stated target-machine interpretation.

### Interrupted execution

If a stage’s first decoded halfword store faults, no later operation is established: neither its repeated store, subsequent state stores, consumer call, nor later stages are guaranteed.

If a dependency does not return, the remainder of the sequence is likewise not guaranteed.

## Uncertainties

The following remain **documented_only**:

- Whether the supplied representation identifies a valid function entry and valid reachable instructions.
- The true entry ABI and provenance of P, B, W, and L.
- Whether represented 64-bit results correspond to actual dependency return types.
- The storage width, address, and aliases of G.
- Memory endianness, alignment requirements, mapped regions, and access permissions.
- Signed-overflow and address-conversion semantics.
- Dependency behavior, ownership conventions, and global-state effects.
- Whether the invalid-instruction boundary corresponds to executable bytes, an intentional trap, or a representation failure.
- Whether repeated stores correspond exactly to distinct machine instructions and externally observed transactions.

Inferred intent is limited to a possible initialization or control sequence. The static evidence does not justify naming a device, protocol, data structure, or reset mechanism.

## Evidence

Established observations come exclusively from the supplied static representation: declared widths, casts, masks, shifts, store expressions, call ordering, explicit argument expressions, and the terminal invalid-instruction warning.

The source also explicitly warns about overlapping global symbols. Accordingly, this contract does not assign an unsupported width or independent storage identity to G.

Store counts, call counts, decoding ranges, and acceptance-example values are arithmetic deductions from that representation, not runtime observations.

**No runtime tests have been performed.** No binary, dependency implementation, execution trace, hardware behavior, or independent calling-convention evidence was available for verification.
