<!-- Trusted provenance: F-fcda8a4fd208; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-fcda8a4fd208

## Purpose

F-fcda8a4fd208 interleaves indirect calls with byte-sized and 32-bit memory accesses. Its visible behavior includes propagating its third explicit argument to two computed addresses, storing scaled dependency results, clearing selected locations, and using unsigned bytes read from memory to select later write destinations.

No application-level purpose is established. Repeated zero stores suggest clearing behavior, but do not establish initialization, resource release, or a particular object layout.

The supplied representation terminates at an invalid-instruction boundary. This contract describes a conditional execution prefix, not a demonstrated normally returning operation. Each later effect requires all preceding calls and accesses to complete.

## Interface

### Explicit inputs

| Input | Width and interpretation | Established use |
|---|---|---|
| First argument | 32 bits; interpretation unspecified | No visible use. |
| Second argument | 32 bits; interpretation unspecified | Supplied twice to the first dependency. |
| Third argument, **T** | Unsigned 32 bits | Forwarded to the first dependency, shifted right by three, and stored unchanged at two computed addresses. |

The unused first argument remains part of the represented interface. Its lack of visible use does not establish a different calling convention.

### Additional state inputs

Execution also depends on:

- An incoming control/link value: adding `0xffe` determines the first indirect-call target.
- An incoming unsigned 32-bit register value, **EntryShiftInput**, shifted logically right by three for the first call.
- An incoming 32-bit register value, **EntryWord**, used by the first direct store.
- Memory at the computed read addresses.
- Dependency results, including a represented unsigned 64-bit result and an additional signed 32-bit register result exposed after a later call.

These are represented entry-state or call-state dependencies, not established ordinary parameters.

### Outputs and preconditions

No return value is represented. Observable outputs are calls and memory effects.

No explicit validation, bounds checking, null checking, status handling, cleanup, or rollback is visible. Reaching an effect requires returning dependencies, callable targets, accessible memory, and valid arithmetic under the applicable execution model. Address width, alignment requirements, and actual calling conventions are documented_only.

The terminal invalid-instruction boundary does not establish a particular exception, recovery mechanism, process termination, or normal return.

## Behavior

### Notation, widths, and conversions

The labels below are documentary names, not source identifiers.

A **four-call tail** comprises four successive indirect calls with no explicit arguments displayed. All four calls occur; when a result is retained, it belongs to the fourth call. Absence of displayed arguments does not establish a zero-argument machine interface.

Retained results **A–AB** are signed 32-bit values unless stated otherwise. **H**, **Y**, and **AA** are unsigned 32-bit values. Some letters are intentionally unused. Forwarded results without a specified interpretation are 32-bit quantities.

- **Scale4(x)** and **Scale16(x)** mean multiplication by four and sixteen.
- **HalfMasked(x)** means unsigned masking with `0x3fffffff`, followed by logical right shift by one.
- **ExtractExpression(x)** means unsigned masking with `0x3fffffff`, followed by logical right shift by 20. Its range is zero through `0x3ff`, not merely zero through `0xff`.
- **CharacterConvert(x)** means conversion of the entire supplied value to the represented eight-bit plain-character type.

Unsigned 32-bit arithmetic wraps modulo \(2^{32}\). This does not resolve signed overflow, signed left-shift validity, or out-of-range conversion to signed 32 bits.

Plain-character signedness and out-of-range signed-character conversion behavior are not supplied. Consequently, CharacterConvert is not unconditionally equivalent to taking the low eight bits. Any concrete stored byte asserted for an out-of-range character conversion requires an established conversion and representation model.

Every word access is 32 bits; every byte access is eight bits. Word endianness is unknown. Repeated offsets do not establish a common object or allocation.

### Phase 1: initial propagation

The first dependency receives, in order, the second argument, the second argument again, T, and EntryShiftInput logically shifted right by three. A four-call tail yields **A**. EntryWord is then written at **Scale4(A) + `0x14`**.

A call receiving Scale4(A), followed by a four-call tail, yields **B**. The routine retains **B4 = Scale4(B)** and writes B4 at **Scale16(B) + `0x14`**. It also retains **T8 = T logically shifted right by three**.

A call receiving B4 and a four-call tail yield **C**. B4 is written at **Scale4(C) + `0x14`**. A call receiving Scale4(C) and another four-call tail yield **D**.

Scale4(D) is evaluated as signed 32-bit multiplication before assignment to unsigned **D4**. D4 is written at **Scale16(D) + `0x14`**. The unsigned destination type does not make an overflowing preceding signed multiplication well-defined.

### Phase 2: masked propagation and a memory-fed call

A call receiving D4 is followed by a call with no explicit arguments. That second call’s result is forwarded to another dependency. A four-call tail yields **E**.

HalfMasked(D4) is written at **Scale4(E) + `0x14`**. A call receiving Scale4(E) and a four-call tail yield **F**. The routine retains **F4 = Scale4(F)** and writes F4 at **Scale16(F) + `0x14`**.

The next dependency receives an unsigned word read from **F4 + `0x14`**, logically shifted right by three. This read address is not generally the address of the immediately preceding store.

A four-call tail yields **G**. F4 is written at **Scale4(G) + `0x14`**. A call receiving Scale4(G) and a four-call tail yield unsigned **H**.

### Phase 3: argument propagation and byte-selected writes

T is written at **T8 × `0x80000` + `0x24`**, using unsigned 32-bit arithmetic.

The routine retains:

- **Base**: H multiplied by sixteen using unsigned 32-bit arithmetic, then converted to signed 32 bits.
- **Q**: HalfMasked(H).

A dependency receives Base and the unsigned value H shifted left by two. Four further calls occur without retained results. Only afterward is T written at **Base + `0x24`**.

A word read at **Base + `0x74`** supplies the next dependency’s argument. A four-call tail yields **J**. The routine reads an unsigned byte at **Base + 1** and writes **CharacterConvert(ExtractExpression(H))** at the address equal to that selector byte plus `0x11`.

A dependency receives J shifted left by two. Another call’s result is forwarded to a further dependency, and a four-call tail yields **K**. A fresh unsigned byte read at **Base + 1** selects a zero-byte store at selector plus `0x11`.

A call receiving K shifted left by two and four further calls follow. The signed expression J shifted left by 23 is then evaluated and retained for the next phase. Afterward:

1. A zero byte is written at **Q + `0x0f`**, using unsigned address arithmetic.
2. **ByteBase** is formed by unsigned 32-bit multiplication of Q by `0x400`, followed by conversion to signed 32 bits.
3. An unsigned byte read at **ByteBase + 1** selects another zero-byte store at selector plus `0x11`.

Selector-based destinations do not include Base or ByteBase. Separate reads cannot be consolidated: intervening calls and earlier writes may change selector memory.

### Phase 4: repeated clearing and the wide result

A dependency receives four arguments: three copies of the retained J-shifted-left-by-23 value, followed by J shifted left by 25. These are signed-shift expressions whose validity is not established for arbitrary J.

Another dependency result is forwarded to a further call. A four-call tail yields **L**. An unsigned byte read at **ByteBase + 1** is retained as **SelectedByte**, then a zero byte is written at **SelectedByte + `0x11`**.

A call receiving L shifted left by two and four further calls precede a zero-word store at **SelectedByte + `0x44`**. This uses the retained selector, not a fresh read. A zero word is then written at absolute address **`0x24`**.

A dependency receives **zero, `0x98`, ByteBase, `0x98`**. Two result-forwarding pairs follow; each pair consists of a call without explicit arguments and a call receiving its result. A four-call tail yields **M**.

A zero word is written at **Scale4(M) + `0x44`**, followed by another zero word at `0x24`.

A call receiving Scale4(M) and a four-call tail yield unsigned 64-bit **WideResult**. Its next two uses have distinct conversion points:

- **First use:** the entire unsigned 64-bit WideResult is converted directly to signed 32 bits. That converted value is then shifted left by two as a signed expression and passed to the next dependency. No preceding extraction of the low 32 bits is established.
- **Second use:** after that dependency returns, WideResult is logically shifted right by 32 as an unsigned 64-bit value. This extracted upper portion is converted to signed 32 bits, multiplied by two, and increased by `0x44`. A zero word is written at the resulting address.

A nonzero upper half makes the entire WideResult exceed signed 32-bit range, even when its low half is small. Deriving the first argument from its low half therefore requires an independently established narrowing model. The second use separately requires valid signed conversion, multiplication, and addition.

Another zero word is written at `0x24`.

The next dependency receives zero, an additional signed 32-bit register result exposed by the preceding dependency, and that additional result shifted left by two. This register result is not established to equal either portion of WideResult.

One call without explicit arguments precedes a call receiving **(Q masked with `0x3fffff`) shifted left by seven**. A four-call tail yields **N**. A zero word is written at **Scale4(N) + `0x44`**, followed by a zero word at `0x24`.

This phase contains four distinct direct zero-word stores to `0x24`. Calls and other effects separate them; they are not contractually redundant.

### Phase 5: further links and masked scaling

A call receiving Scale4(N) and a four-call tail yield **P**. A dependency receiving P shifted left by two yields **R**. A zero word is written at **Scale4(R) + `0x14`**.

A dependency receives R three times, followed by **(Q masked with `0x3fffff`) shifted left by seven**. A four-call tail yields **S**. A zero word is written at **Scale4(S) + `0x14`**.

The following linked propagation occurs:

- A call receiving Scale4(S) and a four-call tail yield **U**. **U4 = Scale4(U)** is written at **Scale16(U) + `0x14`**.
- A call receiving U4 and a four-call tail yield **V**. U4 is written at **Scale4(V) + `0x14`**.
- A call receiving Scale4(V) and a four-call tail yield **W**. Scale4(W) is written at **Scale16(W) + `0x14`**.

A dependency receives Scale4(W) and Scale16(W). Another call’s result is forwarded to a dependency together with **(Q masked with `0x3fffff`) shifted left by four**. A four-call tail yields **X**.

The unsigned value **(Q masked with `0x3fffff`) multiplied by `0x80`** is written at **Scale4(X) + `0x14`**. This product is at most `0x1fffff80` and fits in 32 bits.

A call receiving Scale4(X) and a four-call tail yield unsigned **Y**.

### Phase 6: final character stores and truncated endpoint

**FinalBase** is formed by unsigned 32-bit multiplication of Q by `0x4000000`, followed by conversion to signed 32 bits.

An unsigned byte read at **FinalBase + 1** selects a store at selector plus `0x11`. The stored character is obtained by applying CharacterConvert to **Y shifted left by two using unsigned 32-bit arithmetic**.

A dependency receives that unsigned shifted value, not the converted character. Another dependency result is forwarded to a further call, and a four-call tail yields **Z**.

A fresh byte read at **FinalBase + 1** selects a store of **CharacterConvert(ExtractExpression(Y))** at selector plus `0x11`.

A call receiving Z shifted left by two and a four-call tail yield unsigned **AA**. Another unsigned byte read at **FinalBase + 1** is retained. At this selector plus `0x11`, the routine stores a character through this conversion sequence:

- Convert the entire unsigned AA value to the plain-character type.
- Promote that character value to signed 32-bit integer.
- Multiply the promoted value by four.
- Convert the product to the destination’s plain-character type for the byte store.

The multiplication itself fits in signed 32 bits for any eight-bit character value. Neither character conversion is established as unconditional low-eight-bit truncation. In particular, a small low byte does not make the first conversion representable when the entire AA value is out of range.

The next dependency receives the retained selector zero-extended to 32 bits, not the stored character or a new selector read. Another dependency result is forwarded, and a four-call tail yields **AB**.

An unsigned byte read at **AA × eight + 1**, calculated with unsigned 32-bit arithmetic, selects a store of **CharacterConvert(ExtractExpression(AA))** at selector plus `0x11`.

A dependency receives AB shifted left by two. Four calls without explicit arguments follow. The representation then terminates at the invalid-instruction boundary.

## Side effects

Established direct effects are ordered indirect calls, unsigned byte reads, byte writes, word reads, and word writes.

A destination of **selector + `0x11`** ranges from `0x11` through `0x110`. A word-store starting address of **selector + `0x44`** ranges from `0x44` through `0x143`, with four bytes affected.

These are computed addresses, not established fields. Reads and writes may overlap, including writes to memory later used as selector storage. Their ordering must be preserved.

No allocation, ownership transfer, retention, release, or deallocation is established. Unknown dependencies must not be assumed to preserve globals, reachable memory, or ownership state. No synchronization, atomicity, transaction, or rollback is established.

## Dependencies

Only F-fcda8a4fd208 has a supplied function identity.

Each indirect-call occurrence is designated **F-unknown-〈phase〉-〈ordinal〉**, counting calls in execution order within each Behavior phase. Each member of a four-call tail receives its own ordinal. These opaque IDs identify documentary occurrences, not proven distinct callees.

Established dependency contracts are limited to the displayed argument expressions, result uses, forwarding relationships, call order, and the first target’s dependence on incoming control state.

Repeated target-construction patterns do not establish common identity or behavior. Implicit inputs, calling conventions, errors, memory effects, preservation guarantees, ownership behavior, and whether calls return remain documented_only.

## Acceptance examples

These are conditional static criteria, not executed tests. Unless a criterion addresses unresolved arithmetic, all required accesses, conversions, and signed expressions must be valid.

1. **Initial arguments.** With second argument `0x12345678`, T equal to `0x19`, and EntryShiftInput equal to `0x40` before shifting, the first dependency receives `0x12345678`, `0x12345678`, `0x19`, and eight.

2. **Argument-derived store.** For T equal to `0x19`, T8 is three. The first T store writes `0x19` at `0x180024`.

3. **Distinct scaled addresses.** For F equal to eight, the phase-2 store writes `0x20` at `0x94`. The subsequent word read is at `0x34`.

4. **Fresh selectors.** If the first two phase-3 selector reads yield two and seven, the corresponding destinations are `0x13` and `0x18`. Reusing the first selector is incompatible with the contract.

5. **Retained selector.** If SelectedByte is `0xa0`, the byte clear targets `0xb1` and the later word clear starts at `0xe4`, even if dependencies change the original selector memory.

6. **Representable wide operand.** If the entire WideResult equals three, conversion to signed 32 bits yields three and the next argument is twelve. Its upper-derived zero-word destination is `0x44`; a zero-word store to `0x24` follows.

7. **Nonzero upper half.** If WideResult has upper portion five and lower portion three, its direct conversion to signed 32 bits is out of range. Argument twelve is not unconditional. Under an explicitly assumed modulo-\(2^{32}\), two’s-complement narrowing model, it becomes twelve, and the separate upper-derived address is `0x4e`.

8. **Character boundaries.** ExtractExpression(`0x3ff00000`) is `0x3ff`. A stored byte of `0xff` requires a suitable character-conversion model. Likewise, AA equal to `0x81` produces byte `0x04` under an explicitly assumed unsigned eight-bit plain-character model with modulo conversion, but not as a source-established unconditional rule. AA equal to one produces four without either conversion exceeding the signed or unsigned eight-bit range.

9. **Repeated fixed-address effects.** Completing phase 4 entails four separate direct zero-word stores to `0x24`, preserving intervening calls.

10. **Interrupted execution.** A nonreturning dependency prevents any requirement for subsequent effects. No compensating cleanup is established.

## Uncertainties

The following remain documented_only:

- Reliability of the represented instruction interpretation and control flow.
- Actual interface, implicit inputs, address width, alignment, accessibility, and pointer interpretation.
- Out-of-range conversion of the entire unsigned 64-bit WideResult to signed 32 bits, independently of its low portion.
- Conversion of the extracted upper portion and of the unsigned products forming Base, ByteBase, and FinalBase.
- Signed overflow and signed-shift validity.
- Plain-character signedness, out-of-range character conversions, and resulting byte representations.
- Both character conversions surrounding the final promoted multiplication, including conversion of the entire AA value.
- Word endianness, aliasing, device-memory behavior, and dependency contracts.
- Whether the wide result represents an intentional wide return.
- Runtime consequences of the terminal invalid-instruction boundary.

Clearing or reset intent remains inference only.

## Evidence

The sole evidence is the supplied static representation. It supports the described widths, data dependencies, call/access ordering, direct whole-wide-result conversion, separate upper-portion conversion, character-conversion sequence, and truncated endpoint.

Warnings identify invalid instruction data and overlapping global symbols. Symbolic globals therefore do not establish independent storage objects.

No dependency bodies, caller contracts, verified instruction decoding, character-conversion model, allocation metadata, or runtime observations were supplied. No runtime tests have been performed.
