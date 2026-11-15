<!-- Trusted provenance: F-065fc8ec3ea2; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-065fc8ec3ea2

## Purpose

F-065fc8ec3ea2 describes an ordered sequence of indirect calls, byte accesses, and 32-bit memory accesses. Its inputs include implicit entry machine state, dependency results, and values read from computed addresses.

The supplied static representation does not establish an application-level purpose. Repeated zero stores are consistent with clearing state, but initialization, destruction, allocation, serialization, and device control are not established interpretations.

The represented path ends at an invalid-instruction boundary without a normal return. This does not establish that execution necessarily reaches that boundary: an earlier access may fault, a dependency may not return, or recovered control flow may be incomplete.

## Interface

### Inputs and preconditions

No explicit parameters or return value are declared. The represented operation nevertheless consumes:

- An entry destination base and an entry byte.
- An entry control value used to derive the first indirect-call target.
- An unsigned 32-bit entry word.
- Dependency results interpreted as 32-bit words and, on two occasions, 64-bit quantities.
- Auxiliary machine-state outputs not expressed as conventional return values.
- Memory at dynamically calculated addresses.

A conventional zero-argument invocation therefore does not fully specify the interface. The actual entry ABI, required register contents, and dependency conventions remain **documented_only**.

For execution to proceed through a particular prefix, every reached memory access must be permitted and every reached call target must support the actual machine-state interface. The function does not visibly check these conditions.

### Widths and arithmetic

Byte reads produce unsigned values from 0 through 255. Byte stores retain only the low eight bits of their values. Word accesses are 32 bits. Each 64-bit dependency result is separated into low and high 32-bit halves where described below.

The behavioral formulas use 32-bit bit-vector arithmetic: multiplication, left shifts, and address addition retain the low 32 bits. Right shifts of unsigned quantities are logical. This interpretation captures the represented widths; it does not establish portable signed-overflow semantics for expressions presented with signed types.

The following descriptive quantities are used:

| Quantity | Definition |
|---|---|
| **Top** | Entry word logically shifted right by 25; range 0–127 |
| **Packed** | Top multiplied by \(2^{23}\) |
| **Region** | Packed logically shifted right by three; equivalently Top multiplied by \(2^{20}\) |
| **Four(x)** | Four times x, reduced to 32 bits |
| **Sixteen(x)** | Sixteen times x, reduced to 32 bits |
| **HighField(x)** | \((x \mathbin{\&} \texttt{0x3fffffff}) \gg 20\) |
| **HalfField(x)** | \((x \mathbin{\&} \texttt{0x3fffffff}) \gg 1\) |

A word access covers four consecutive bytes beginning at its calculated address. Byte order, alignment requirements, and fault semantics are not established.

### Outputs and ownership

No normal result, success flag, error code, or ownership-transfer result is established.

The visible operations do not identify allocation, release, retention, or reference counting. A value passed to a dependency must not be interpreted as an ownership transfer without additional evidence.

## Behavior

### Ordering and dependency model

The following phases describe the represented order of direct memory effects and their relevant call relationships. They are conditional on preceding execution reaching each phase.

All unresolved dependencies are designated **F-UNRESOLVED**, an opaque identifier for an unresolved target set, not a claim that every call reaches one function. Intervening calls remain part of the behavior even when their full argument lists are unavailable here. Their effects may change later memory observations, affect machine state, fault, or prevent further execution.

Repeated reads of the same address are fresh observations. Saved payloads and auxiliary values below describe represented data dependencies, not an independently verified machine-level preservation convention.

### Entry propagation and the shared auxiliary value

The first direct effect is a byte store to the entry destination base plus one, using the entry byte. It occurs before any dependency call.

An entry-control-dependent call follows. A later call returns a 32-bit value that is forwarded to another dependency. After additional calls, a result supplies an initial base equal to Four(result). A word is read at that base plus `0x74`; its low byte is then stored at the same base plus one. A subsequent call receives the initial base.

After further calls, a single represented auxiliary value, called **Auxiliary A**, supplies both:

- The destination of a word store: Four(Auxiliary A) plus `0x14`.
- The argument of the following relevant dependency call.

That store writes Top multiplied by \(2^{19}\).

The next relevant 64-bit dependency result is passed onward as three arguments: low half, high half, low half.

After intervening calls, a later 32-bit result selects another word destination, Four(result) plus `0x14`. The stored value is **the same represented Auxiliary A**, shifted left by three. The following call receives Four(result).

Thus the earlier destination calculation, the earlier call argument, and this later stored value share one represented symbolic input. The representation does not introduce a replacement auxiliary value between them. Whether that shared value’s lifetime was accurately recovered across the intervening calls remains unresolved; unknown dependency effects alone do not establish that the later use consumes a fresh value.

### Region-derived stores

Following additional calls, a word is stored at:

\[
\text{Region}\times\texttt{0x200000}+\texttt{0x14},
\]

with value Region multiplied by `0x80000`. Region is a multiple of \(2^{20}\), so both products reduce to zero at 32 bits. This is therefore a zero word store at absolute address `0x14`. The next relevant call receives three copies of the zero product.

A later result selects a destination at Four(result) plus `0x14`, where Region is stored. The scaled result is passed onward.

### First byte-directed payload and linked word stores

A subsequent result becomes **First Payload**, with **First Scaled** equal to Four(First Payload). A byte is read at Region plus one. Its unsigned value selects an absolute byte destination at that value plus `0x11`; the low byte of First Scaled is stored there.

The following call receives, in order:

1. The observed byte logically shifted right by three.
2. First Scaled.
3. Four(Region).

After further calls, the byte at Region plus one is read again. This observation becomes **Selector**. The low byte of HighField(First Payload) is stored at Selector plus `0x11`. A dependency receives four times a recently returned result.

Following more calls, First Scaled is stored as a word at Four(Selector) plus `0x14`. The following relevant call receives Selector three times, then Selector logically shifted right by one.

Another result selects a destination at Four(result) plus `0x14`, where First Scaled is stored. The scaled result is passed onward.

Three subsequent result-derived links produce these ordered effects, with calls between link stages:

- **First Link:** Four(First Link) is stored at Sixteen(First Link) plus `0x14` and passed onward. A later result selects a second word destination at Four(result) plus `0x14`, receiving the same scaled link value.
- **Second Link:** Four(Second Link) is stored at Sixteen(Second Link) plus `0x14` and passed onward. A dependency result is forwarded to another dependency. A later word store writes HalfField(Four(Second Link)) at Four(a later result) plus `0x14`.
- **Third Link:** Four(Third Link) is stored at Sixteen(Third Link) plus `0x14`. A word is then read at Four(Third Link) plus `0x14`, and that observed word, logically shifted right by three, is passed to a dependency. These read and write addresses generally differ. After intervening calls, Four(Third Link) is stored at Four(a later result) plus `0x14`.

The scaled destination results are passed onward at the represented call sites. Neither a prior store nor a saved payload substitutes for the explicit Third Link word read.

### Later payload and byte clearing

A new result becomes **Later Payload**. Selector is stored as a word at:

\[
(\text{Selector}\gg3)\times\texttt{0x80000}+\texttt{0x24}.
\]

Two retained quantities govern later accesses:

- **Later Base** is Sixteen(Later Payload).
- **Reduced** is HalfField(Later Payload).

A dependency receives Later Base and Four(Later Payload).

After intervening calls, Selector is stored at Later Base plus `0x24`. A word is then read at Later Base plus `0x74` and passed to a dependency.

A later result becomes **Shift Source**. A byte is read at Later Base plus one, and the low byte of HighField(Later Payload) is stored at that observed byte plus `0x11`. Four(Shift Source) is passed onward.

Following additional calls, including a result-forwarding pair, Later Base plus one is read again. Zero is stored at this newly observed byte plus `0x11`. Four times the latest relevant call result is passed to a dependency.

After another call sequence, zero is stored as a byte at Reduced plus `0x0f`. **Clear Base** is Reduced multiplied by `0x400`. A byte is then read at Clear Base plus one, and zero is stored at that observed byte plus `0x11`. No dependency call separates these two zero stores; the intervening byte read must nevertheless observe memory after the first store.

The subsequent relevant call receives Shift Source shifted left by 23 three times, followed by Shift Source shifted left by 25, all at 32-bit width.

After further calls, Clear Base plus one is read again. This observation becomes **Clear Selector**. Zero is stored at Clear Selector plus `0x11`, and four times a newly returned result is passed onward.

### Word clearing and repeated absolute writes

Following more calls, a zero word is stored at Clear Selector plus `0x44`, then a zero word at absolute address `0x24`. The next relevant call receives zero, `0x98`, Clear Base, and `0x98`.

After several calls, including result-forwarding operations, a result selects a zero word store at Four(result) plus `0x44`. A second zero word store immediately follows at absolute `0x24`. The scaled result is passed onward.

A second 64-bit result is then obtained. A dependency is called with Four(its low half). After that call, a zero word is stored at twice the retained high half plus `0x44`, followed by another zero word at absolute `0x24`.

A later call receives zero, another represented auxiliary output, and four times that output. This auxiliary output is distinct in the representation from Auxiliary A; no equality or inequality between their numerical values is established. Another call receives:

\[
(\text{Reduced}\mathbin{\&}\texttt{0x3fffff})\times128.
\]

After more calls, another result selects a zero word store at Four(result) plus `0x44`, immediately followed by the fourth explicit zero word store at absolute `0x24`. The scaled result is passed onward.

These four absolute `0x24` stores are separate observable writes, not merely a final requirement that the location contain zero.

### Remaining word updates

A result is scaled by four and passed to another dependency. That dependency’s result becomes an index. Zero is stored at Four(index) plus `0x14`. The following call receives the index three times, then \((\text{Reduced}\mathbin{\&}\texttt{0x3fffff})\times128\).

After further calls, another result selects a zero word store at Four(result) plus `0x14`; its scaled value is passed onward.

Two more link stages follow:

- Four(**Fourth Link**) is stored at Sixteen(Fourth Link) plus `0x14` and passed onward. A later result selects another destination at Four(result) plus `0x14`, receiving Four(Fourth Link).
- Four(**Fifth Link**) is stored at Sixteen(Fifth Link) plus `0x14`. The next relevant call receives Four(Fifth Link) and Sixteen(Fifth Link).

A subsequent dependency result is forwarded together with \((\text{Reduced}\mathbin{\&}\texttt{0x3fffff})\times16\).

Following more calls, \((\text{Reduced}\mathbin{\&}\texttt{0x3fffff})\times128\) is stored at Four(a returned result) plus `0x14`. The scaled destination result is passed onward.

### Final byte-directed sequence

A result becomes **Penultimate Payload**. **Final Base** is Reduced multiplied by `0x4000000`, reduced to 32 bits.

A byte is read at Final Base plus one. The low byte of Four(Penultimate Payload) is stored at that observed byte plus `0x11`, and the scaled payload is passed onward.

After result forwarding and additional calls, Final Base plus one is read again. The low byte of HighField(Penultimate Payload) is stored at the newly observed byte plus `0x11`. Four times the latest relevant result is passed onward.

A new result becomes **Final Payload**. Final Base plus one is read a third time. The low byte of Four(Final Payload) is stored at that observation plus `0x11`. The observed byte itself is passed to a dependency.

Following additional calls, a byte is read at eight times Final Payload plus one, rather than at Final Base plus one. The low byte of HighField(Final Payload) is stored at this observation plus `0x11`. Four times the latest relevant result is passed onward.

Several indirect calls follow. The supplied representation then terminates at an invalid-instruction boundary without a normal-return path.

## Side effects

Direct side effects are byte and word writes at computed addresses, including the zero word store at absolute `0x14` and four explicit zero word stores at absolute `0x24`.

An unsigned byte used as a destination selector produces an absolute byte-store address from `0x11` through `0x110`. It is not shown as an offset within a separately established object. Clear Selector plus `0x44` can likewise select an unaligned word address.

Aliasing is unrestricted by the evidence. Direct stores may affect subsequent reads, and dependencies may modify memory between accesses. The two zero stores involving Reduced and Clear Base have no intervening call, but their intervening read still requires correct alias-sensitive ordering.

No bounds checks, null checks, alignment checks, address validation, rollback, or error recovery are visible. Fault delivery and the visibility of preceding writes remain environmental concerns.

The memory model does not establish atomicity, synchronization, or hardware-register semantics. Overlapping-global annotations do not establish independent storage for the absolute-address writes.

## Dependencies

**F-UNRESOLVED** covers all unresolved call targets.

The first target depends on entry control state. Later targets are formed from represented control-location values and offsets. The supplied dependency map resolves no external callee identity.

Observed call relationships include result forwarding, scaled arguments, repeated arguments, split 64-bit results, and auxiliary-state consumption. Calls shown without explicit arguments do not prove the absence of implicit machine-register inputs.

No dependency preservation, ownership, lifetime, allocation, release, or error-handling contract is established. In particular:

- Auxiliary A is reused symbolically across the early calls; its recovered lifetime remains uncertain.
- The later auxiliary output has separate represented provenance.
- Unknown callee behavior does not authorize replacing either represented data dependency with a newly invented value.

Exact target identities and a complete independent call-interface specification remain **documented_only**.

## Acceptance examples

These are conditional static examples, not runtime observations.

1. **Entry order.** If the entry destination is writable, its byte at base plus one is written before the first dependency call.

2. **Shared auxiliary dependency.** If Auxiliary A is represented as five, the early word destination is `0x28`, the associated call argument is five, and the later Auxiliary-A-derived store value is 40. A separately supplied destination result of three places that later word at `0x20`. Introducing a fresh auxiliary value for the later store does not match the represented dependency. This example does not verify preservation across actual calls.

3. **Arithmetic collapse.** For every entry word, Region is a multiple of \(2^{20}\). The Region-derived product store writes a zero word at absolute `0x14` under 32-bit arithmetic.

4. **Fresh selector.** If Region plus one yields `0x08` initially and `0x20` on its later read, the corresponding byte-store destinations are `0x19` and `0x31`. Reusing the first observation is incorrect.

5. **Byte truncation.** For a payload of `0x3fffffff`, HighField is `0x3ff`, but its byte store writes `0xff`.

6. **Distinct linked addresses.** If Third Link is three, its direct word store writes 12 at `0x44`; the following word read is at `0x20`. The forwarded argument depends on the word actually read at `0x20`.

7. **Repeated clearing.** If a dependency changes memory at `0x24` between explicit zero stores, each later zero store still occurs when reached.

8. **Final read-source change.** If Final Base and eight times Final Payload differ, the last byte selector comes from the latter address plus one, not from the preceding three reads’ address.

9. **Failure and termination.** An inaccessible computed address establishes neither rollback nor successful completion. A conventional success return is not supported by the represented terminal boundary.

## Uncertainties

The following remain **documented_only**:

- Actual ABI, function boundary, and executable reachability.
- Dependency identities, implicit inputs, effects, and return behavior.
- Accurate lifetime recovery for Auxiliary A across intervening calls.
- Provenance and preservation rules for auxiliary outputs generally.
- Exact instruction semantics where signed expressions obscure width behavior.
- Endianness, alignment, permissions, address mapping, and fault handling.
- Concurrency, atomicity, barriers, and external observers.
- Application-level meaning, resource ownership, and object layout.

Offsets `0x14`, `0x24`, `0x44`, and `0x74` establish access formulas, not a verified structure definition.

## Evidence

**Established static evidence:** the supplied representation contains ordered byte and 32-bit accesses, unsigned byte-directed addressing, indirect calls, two split 64-bit results, repeated absolute zero stores, and an invalid-instruction terminal warning. The same auxiliary-output symbol supplies the early scaled destination, its following call argument, and the later left-shifted word value.

**Inferred intent:** repeated clearing and scaled writes may manipulate structured state. No particular subsystem or lifecycle operation is established.

**Evidence limits:** the supplied material warns of unreliable instruction/control-flow recovery and overlapping global symbols. Symbolic reuse establishes a dependency in the representation, not independently verified lifetime preservation during execution.

No implementation access, dependency inspection, runtime execution, or runtime tests have been performed.
