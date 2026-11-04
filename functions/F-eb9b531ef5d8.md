<!-- Trusted provenance: F-eb9b531ef5d8; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-eb9b531ef5d8

## Purpose

F-eb9b531ef5d8 initializes five consecutive 32-bit words in caller-supplied storage, performs two 16-bit memory reads, and then performs additional writes selected by whether its second explicit input is zero. On normal completion, it returns its fifth explicit input unchanged.

The zero-input path is **not** a safe null-input shortcut: the first 16-bit read occurs before the zero comparison and uses an address four bytes beyond the second input. Thus, a zero second input still requires a readable location at address `0x00000004` to reach the conditional writes.

The operational purpose of these writes is unknown. Initialization, low-level state capture, or platform setup are possible interpretations, but none is required by the established behavior.

## Interface

### Explicit inputs

| Position | Contract |
|---|---|
| First | A 32-bit unsigned value. Its highest four bits supply the first destination word. |
| Second | A value represented as a signed 32-bit integer and used as an address base. It also controls the conditional path through an exact comparison with zero. |
| Third | A pointer to writable storage containing at least five consecutive 32-bit words, subject to the accessibility and alignment requirements of the execution environment. Its address is also converted to a 32-bit unsigned value and stored in that storage. |
| Fourth | A value represented as a signed 32-bit integer. Subtracting 31 supplies the bit pattern written to the third destination word, subject to the arithmetic qualification below. |
| Fifth | An opaque 64-bit value returned unchanged if execution completes normally. |

No explicit buffer length, error-output pointer, ownership flag, or execution-context argument is present.

### Implicit inputs

Three additional incoming values participate in the behavior:

- **I4:** an incoming value represented as signed 32-bit, used to derive a memory base.
- **I5:** an incoming 32-bit value written to the fourth destination word.
- **I7:** an incoming 32-bit value written to the fifth destination word and, on the zero-input path, to a global destination.

Their availability is established by the supplied representation, but their calling-convention status is not. A usable external interface must account for them; the five explicit arguments alone do not determine all observable behavior.

Define **B** as the base derived by multiplying I4 by `0x10000000`. The multiplication is represented as signed 32-bit arithmetic. Overflow and address interpretation are qualified under **Uncertainties**.

### Destination layout

Offsets below are relative to the third explicit input:

| Byte offset | Write width | Value written |
|---:|---:|---|
| `0x00` | 32 bits | First input shifted right by 28 bits; result is in the range 0–15 |
| `0x04` | 32 bits | Destination pointer converted to a 32-bit unsigned value |
| `0x08` | 32 bits | Fourth input minus 31 |
| `0x0C` | 32 bits | I5 |
| `0x10` | 32 bits | I7 |

These writes cover 20 bytes. No write beyond this five-word region is implied by the destination-initialization portion alone.

The layout specifies word offsets and widths, not byte order. Endianness is not established by the supplied material.

### Return and completion

Normal completion returns the fifth explicit input as the same opaque 64-bit value. There is no visible success/failure encoding in that return value.

No normal error result, explicit exception handler, rollback, or retry is shown. A memory-access failure or arithmetic behavior outside the established execution model may prevent normal completion.

## Behavior

### 1. Initialize destination storage

The five destination writes occur in ascending offset order:

1. Store the high four bits of the first input at offset `0x00`.
2. Store the destination pointer’s 32-bit unsigned representation at offset `0x04`.
3. Store the fourth input minus 31 at offset `0x08`.
4. Store I5 at offset `0x0C`.
5. Store I7 at offset `0x10`.

The first input is unsigned, so its right shift is a logical shift. No lower input bits contribute to the first stored value.

The pointer written at offset `0x04` is the pointer value itself after conversion, not a value loaded from the destination. If the execution environment has pointers wider than 32 bits, the exact conversion result requires an external ABI definition.

These stores precede both 16-bit reads. Consequently, if the destination aliases either later read location, the read may observe bytes written during initialization.

### 2. Obtain two 16-bit values

After deriving B, the function performs the following reads in this order:

1. Read an unsigned 16-bit value at the second input’s address plus `0x04`. Call this value **H**.
2. Read an unsigned 16-bit value at B plus `0x28`. Call this value **Q**.

Both values are interpreted as unsigned. When subsequently written as 32-bit quantities, their values are zero-extended: the upper 16 bits are zero.

Both reads precede the comparison of the second input with zero. In particular:

- A zero second input does not suppress the read at address `0x00000004`.
- A zero second input does not suppress the read at B plus `0x28`.
- On the nonzero path, H is still read even though no later shown write uses it.
- On the zero path, Q is read and used in an additional write before the final write changes to a fixed value.

A replacement that skips either read because its value appears unused on a selected path would not preserve the established access sequence.

### 3. Select writes using the second input

For describing global effects, **G1** through **G5** identify the five named global destinations in write order. These labels do not assert that the destinations are distinct, disjoint, or semantically independent.

#### Nonzero second input

If the second input is nonzero, the function writes Q as a 32-bit value to B plus `0x30`.

No writes to G1 through G5 are shown on this path. No write to B plus `0x3B` is shown.

The earlier read of H remains part of this path’s behavior despite its value not being used afterward.

#### Zero second input

If the second input is zero, the function performs these writes in order:

1. Write B minus 5 to G1.
2. Write Q as a 32-bit value to B plus `0x3B`.
3. Write `0xFFFFFB00` to G2.
4. Write H, zero-extended to 32 bits, to G3.
5. Write B to G4.
6. Write I7 to G5.
7. Write `0x0000008D` as a 32-bit value to absolute address `0x00000030`.

The final destination on this path is absolute address `0x30`, not B plus `0x30`.

The write at B plus `0x3B` has a four-byte width. If B is word-aligned, this destination is not word-aligned. Whether that write succeeds, faults, is split, or has device-specific behavior depends on the execution environment.

G1 and G4 receive 32-bit-derived values. G2 receives the stated 32-bit constant. G3 and G5 receive 32-bit values. The exact backing storage widths and access semantics of the named global destinations are not independently established.

### 4. Return

After the selected final 32-bit write completes, the function returns the fifth input unchanged.

The returned value does not report whether the destination or global state has a meaningful interpretation. It also does not summarize values read from memory.

## Side effects

### Memory-access ordering

The established sequence is:

- Five destination-word writes.
- Read H.
- Read Q.
- Conditional global and auxiliary writes, if the second input is zero.
- One final 32-bit write.
- Normal return.

This ordering is significant for aliasing, partial completion, and memory-mapped locations. It does not establish hardware barriers, inter-thread synchronization, cache behavior, or atomicity.

An access that faults can leave earlier writes visible. The function has no shown mechanism to restore previous contents.

### Aliasing and overlapping storage

No non-aliasing guarantee is supplied. The destination region, the two read locations, the B-relative write locations, the absolute final-write location, and the named globals may overlap where the address model permits.

Consequences include:

- Destination initialization may change H or Q before either is read.
- Later writes may overwrite some or all of the initialized destination words.
- A global write may affect storage also exposed through another global symbol.
- A later overlapping write may replace bytes written earlier.

The supplied material explicitly warns that some global symbols overlap smaller symbols. Therefore, G1 through G5 must not be modeled as five isolated abstract variables without additional layout evidence.

H and Q are loaded before the conditional writes. Their later uses consume those loaded values; no second read of either location is shown.

### Ownership and lifetime

The function receives storage addresses and writes through them. No allocation, deallocation, ownership transfer, reference-count operation, or lifetime extension is shown.

The destination must remain usable for all applicable writes. Its address being stored within the destination does not itself create an ownership relationship or guarantee that the address remains valid after return.

### Preconditions for normal completion

Normal completion requires, at minimum:

- All five destination writes to be permitted.
- Both 16-bit reads to be permitted, including the read at address `0x04` when the second input is zero.
- Every write selected by the branch to be permitted.
- The execution environment to support the required access alignments and address calculations.
- A defined interpretation for the implicit inputs and the arithmetic used to derive B.

These are execution requirements, not checks performed by the function.

## Dependencies

No function calls are visible in F-eb9b531ef5d8. No additional opaque function IDs are available or needed for its directly shown behavior.

Its external dependencies are instead:

- The calling convention or entry context that supplies I4, I5, and I7.
- The process or machine address model.
- Memory accessibility, endianness, alignment rules, and device-access semantics.
- The addresses, widths, and overlap relationships of G1 through G5.
- The arithmetic interpretation used for signed operations and address formation.

There is no evidence of a callee that validates addresses, preserves globals, or manages memory. No such guarantees are assumed.

## Acceptance examples

These are conditional contract examples, not results of executed tests. Unless stated otherwise, assume accessible, suitably supported memory; no unintended aliasing; and arithmetic within established representable ranges.

### Nonzero path with destination initialization

Given:

- First input: `0xD2345678`.
- Fourth input: 50.
- I4: 1, producing B = `0x10000000`.
- I5: `0x11223344`.
- I7: `0x55667788`.
- A nonzero second input.
- H = `0x1234`.
- Q = `0xABCD`.

The five destination words are written, in order, as:

| Offset | Value |
|---:|---:|
| `0x00` | `0x0000000D` |
| `0x04` | Destination pointer’s 32-bit unsigned representation |
| `0x08` | `0x00000013` |
| `0x0C` | `0x11223344` |
| `0x10` | `0x55667788` |

H is read before Q. The final write places `0x0000ABCD` at address `0x10000030`. The fifth input is returned unchanged.

### Zero path with readable low memory

Given a zero second input, I4 = 1, readable address `0x04` containing H = `0x1234`, and Q = `0xABCD` at address `0x10000028`:

- Destination initialization occurs first.
- Both 16-bit reads occur.
- G1 receives `0x0FFFFFFB`.
- Address `0x1000003B` receives the 32-bit value `0x0000ABCD`.
- G2 receives `0xFFFFFB00`.
- G3 receives `0x00001234`.
- G4 receives `0x10000000`.
- G5 receives I7.
- Absolute address `0x00000030` receives `0x0000008D`.

This example requires support for the potentially unaligned four-byte write.

### Zero input with inaccessible address `0x04`

If the five destination stores complete but the subsequent read at address `0x04` faults, the shown sequence does not reach the Q read, conditional writes, final write, or normal return. Earlier destination changes are not rolled back by any shown behavior.

### Boundary values in destination fields

A first input of `0x0FFFFFFF` produces zero in the first destination word; `0xFFFFFFFF` produces 15.

A fourth input of 31 produces zero in the third destination word. A fourth input of 30 produces the signed result −1, stored in an unsigned 32-bit destination as `0xFFFFFFFF`.

### Aliased read location

If the location used to read H overlaps destination bytes modified during initialization, H must be determined from memory after all five initialization stores. Its value is not necessarily the value present on entry. The exact result depends on the overlap and endianness.

## Uncertainties

- **documented_only — implicit-input convention:** The origin and required entry-state setup for I4, I5, and I7 are not established.
- **documented_only — signed multiplication:** B is represented as a signed 32-bit product. Modulo-`2³²` multiplication must not be assumed for overflowing inputs solely from this representation.
- **documented_only — subtraction and address arithmetic:** Signed overflow in the fourth input minus 31, and overflow or underflow in address calculations and B minus 5, lack an independently established execution rule.
- **documented_only — pointer model:** Address width, pointer conversion rules, and interpretation of signed address-bearing values are not fully established.
- **documented_only — global layout:** Exact global addresses, backing widths, and overlapping-symbol relationships are unavailable.
- **documented_only — machine access behavior:** Endianness, unaligned access handling, fault semantics, and memory-mapped side effects are unspecified.
- **Inferred intent:** The function may manipulate low-level platform state, but the evidence does not establish a subsystem, device, or state-transition meaning.

## Evidence

This contract is based exclusively on static inspection of the supplied representation.

Established static evidence includes the five destination writes and their order; the two unsigned 16-bit reads before the zero comparison; both branch-specific write sequences; the absolute final destination on the zero path; and the unchanged 64-bit return value. The representation also exposes three implicit incoming values and warns of overlapping global symbols.

The distinction between source-level arithmetic representation and actual machine overflow behavior remains unresolved. Global semantic names are not used as proof of runtime purpose.

No runtime tests have been performed. The acceptance examples are conditional predictions from the static evidence, not observed executions.
