<!-- Trusted provenance: F-f6dabb4f53c5; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-f6dabb4f53c5

## Purpose

The supplied static representation describes a routine that interleaves indirect calls with byte-sized and 32-bit memory accesses. Values obtained from calls and memory are repeatedly scaled, masked, shifted, and reused as addresses or call arguments.

No application-level purpose is established. In particular, the evidence does not establish that this routine initializes an object, allocates storage, decodes a format, or operates on a conventional collection. Those interpretations would require information about the indirect callees and the executable environment.

The representation includes a bad-instruction warning and terminates at an instruction-decoding failure. Consequently, this contract describes the visible execution prefix, not a verified complete function. It does not establish a normal return or a reliable terminal operation.

## Interface

### Explicit inputs

Two 32-bit arguments are declared.

- The first argument has no visible use in the supplied representation. This does not prove that the original executable ignores it: indirect-call conventions and instruction recovery are unresolved.
- The second argument is passed to the first visible indirect call.

No signedness, pointer interpretation, ownership transfer, or validation requirement is established for either argument.

### Additional execution inputs

The visible behavior also depends on:

- An incoming machine value used to derive the first call target.
- A separate incoming 32-bit machine value used both as an address and as the source of a byte.
- Results from indirect calls, including apparent 64-bit results.
- Secondary machine-register outputs from calls that are not represented as ordinary return values.
- Existing memory at computed addresses.
- The validity and behavior of every indirect call target.

These dependencies prevent the explicit two-argument interface from being a complete standalone specification.

### Output and termination

The declared return type is void. No explicit returned data is established.

Observable outputs consist of memory writes and effects of indirect calls. The visible prefix ends at an undecodable instruction. A faithful implementation must not silently reinterpret that endpoint as successful return, a particular exception, or a particular process-termination operation.

### Width and addressing model

The representation uses:

- 8-bit reads and writes.
- 32-bit reads, writes, addresses, and most call values.
- Apparent 64-bit call results, subsequently separated into low and high 32-bit halves.

Byte stores retain the low eight bits. Unsigned right shifts are logical. Several left shifts and multiplications are represented using signed intermediate types, so their exact language-level semantics are not established for negative values or overflow.

Where this contract discusses wrapped arithmetic, that is a **conditional 32-bit machine interpretation**: retain the low 32 bits of arithmetic results. It is not evidence that the represented signed expressions have defined overflow behavior.

Endianness, alignment requirements, address-space mapping, and permitted access widths remain unknown.

## Behavior

### General execution rules

The visible prefix is linear: no conditional branch, loop, explicit validation failure, or early return is shown. Each later operation is reachable only if all preceding calls and accesses permit execution to continue.

The order of explicit accesses matters. Intervening calls cannot be treated as inert: they may change memory, produce additional externally visible effects, or fail to return.

Descriptions below name values by their role in the contract, not by source identifiers. A retained value means the value shown as carried forward by the representation; it does not imply that its backing memory remains unchanged.

### 1. Entry calls and initial selector writes

The first indirect call receives the second explicit argument. Its target depends on incoming machine state.

A subsequent call supplies a 32-bit value used later as a memory base. That value is passed to another call. After further calls, another returned value is obtained.

Before that latter value, scaled by four, is passed onward:

1. Read one byte at offset 1 from the incoming address-bearing machine value.
2. Zero-extend the byte to obtain a selector.
3. Write the high byte of the incoming 32-bit value to address `selector + 0x11`.

After another call sequence, a returned value is multiplied by four and retained as a scaled base. Then:

1. Read a 32-bit word at offset 4 from the earlier call-derived memory base.
2. Read a byte at offset 1 from that word interpreted as an address.
3. Write zero at the resulting selector plus `0x11`.
4. Call an indirect dependency with three equal arguments, each the scaled base.

Following more calls, the previously loaded word is logically shifted right by two and used as an address base. A byte is read at offset 1 from that shifted address. That byte becomes a new retained selector, and zero is written at its address plus `0x11`.

These are distinct reads and writes. The latter selector is not necessarily the selector from the preceding zero write.

### 2. Word extraction and low-byte feedback

After additional calls, the routine:

1. Reads a 32-bit word at offset 4 from the retained scaled base.
2. Reads another 32-bit word at offset `0x74` from the retained selector.
3. Writes the low byte of the second word at selector offset 1.

An indirect call then receives, in order:

- The second word.
- The scaled base shifted left by 19.
- The same shifted value again.
- A newer call result scaled by four.

A subsequent call result is passed onward together with the second word logically shifted right by two.

The first word read in this phase is logically shifted right by 22 and retained. It therefore contains at most ten significant bits.

Later, another call-derived value is scaled by four. The routine reads a 32-bit word at its offset `0x74`, writes that word’s low byte at offset 1 of the same scaled base, and passes the base to a call.

After further calls, a secondary machine-register output supplies a destination index. The previously loaded word is written to address `4 × index + 0x14`. That index is then passed to another call.

The association between this secondary output and a particular callee’s formal return convention is unknown.

### 3. Packed values and scaled-index stores

A call-produced value is forwarded with two equal additional arguments. Each equal argument is formed by taking the low nine bits of the earlier ten-bit extraction and shifting them left by 21.

A later call-derived index receives a 32-bit store at `4 × index + 0x14`. The stored value is the ten-bit extraction shifted left by 23. Under 32-bit wrapping, this store discards the extraction’s highest bit; the earlier duplicated arguments retain its low nine bits at a different bit position.

A new returned value then produces:

- A 32-bit store at `16 × value + 0x14`, containing `4 × value`.
- A retained masked value equal to the return value with its top two bits cleared.
- A retained quotient equal to that masked value logically shifted right by two.

The masked value is passed to a call. A later returned index selects address `4 × index + 0x14`, where the masked value is stored. The scaled index is then passed onward.

A subsequent return value drives two more writes:

1. Write the low byte of `8 × retained index` at offset 1 from the earlier word loaded through offset `0x74`.
2. Write four times the new return value at address `8 × retained index + 0x34`.

The scaled new return value is passed to a call.

### 4. Split results and overlapping updates

The preceding return value, masked to 30 bits, becomes another retained base.

Following intervening calls, an apparent 64-bit result supplies two halves:

1. Write the low byte of the retained eightfold index at offset 1 from the masked base.
2. Write the result’s high 32 bits at the same `eightfold index + 0x34` destination used in the preceding phase.
3. Pass the result’s low 32 bits, scaled by four, to a call.

Thus the second 32-bit store supersedes the earlier store at that exact computed destination, if execution reaches it. Intervening calls may observe or modify the first stored value.

After more calls:

- A retained index is scaled by 16.
- Its low byte is written to an address formed from bits 20–29 of an earlier 30-bit masked value, plus `0x11`.
- The low byte of the earlier fourfold return value is written at offset `0x0d` from the sixteenfold index.

A call receives two equal arguments, each a newer return value scaled by four. Another call returns a value that is forwarded together with its logical right shift by three.

After a further call sequence, the routine writes:

1. The low byte of the sixteenfold index to absolute address `0x11`.
2. The same earlier fourfold value’s low byte again at the sixteenfold index plus `0x0d`.

The second write repeats an explicit destination and value across intervening calls; it is not safely removable.

### 5. Additional indexed fields and selector feedback

Following further calls:

1. Store the earlier fourfold value at offset `0x14` from the retained 30-bit masked base.
2. Store the low byte of twice that base at offset `0x0d`.

A new returned value determines:

- A base equal to 32 times that value.
- A payload equal to that value shifted left by seven.
- A reduced selector equal to bits 17–26 of that value.

After intervening calls, another return value is obtained. The routine writes:

1. The payload at base offset `0x14`.
2. The low byte of four times the newer return value at base offset `0x0d`.

The latter scaled return value is passed onward.

After another call sequence:

1. Write the low byte of twice the earlier retained quotient at `reduced selector + 7`.
2. Read a byte at offset 1 from the 32-fold base.
3. Write the low byte of the reduced selector at that byte-selected address plus `0x11`.

A call then receives three equal arguments derived by shifting the newer return value left by 21.

Following additional calls, the byte at base offset 1 is read again. Zero is written at that freshly selected address plus `0x11`. The fresh read is significant: intervening calls may have changed the selector.

### 6. Low-address state and secondary outputs

After another call sequence, the freshly read selector is doubled and retained as a small base. A previously returned unsigned value, logically shifted right by three, supplies a retained byte payload.

The routine writes:

1. That payload at offset `0x0d` from the preceding 32-fold base.
2. The selector itself to absolute address `0x0d`.

An indirect call receives the doubled selector, `0x98`, and `0x260`.

After another call, a secondary machine-register value is logically shifted right by three and passed onward. No stable ABI for this secondary output is established.

Following more calls, a returned value is scaled by four and passed to a dependency that itself returns a value. Before forwarding four times that new result, the routine writes:

1. The retained byte payload at doubled-selector offset `0x0d`.
2. The byte `0xc0` at absolute address `0x0d`.

Therefore, the two explicit writes to absolute address `0x0d` are ordered: selector first, `0xc0` later. Callees may observe or modify that location between them.

### 7. Linked word accesses and paired return halves

A later returned value drives a store of four times that value at doubled-selector offset `0x24`. The routine then reads a 32-bit word at doubled-selector offset `0x74`.

A call receives four arguments:

- The loaded word.
- Twice the loaded word.
- The latest returned value scaled by 32.
- The loaded word with its top bit cleared, logically shifted right by two.

After intervening calls, the 32-fold value is written at offset `0x24` from the loaded word interpreted as an address. A word is then read at offset `0x74` from that same address and passed to a call.

A later apparent 64-bit result is split into halves:

1. Store an earlier unsigned value shifted right by three at `4 × high half + 0x14`.
2. Form four times the low half.
3. Call a dependency with that scaled low half repeated three times, followed by the high half logically shifted right by three.

Another apparent 64-bit result is forwarded as four arguments: low half, high half, low half again, and high half shifted right by three.

Following more calls, the retained scaled low half is written at `4 × newly returned index + 0x14`; that scaled index is then passed to a call.

### 8. Final visible selector writes

A later unsigned return value is scaled by four.

The routine reads a byte at offset 1 from the earlier word-derived address and writes the low byte of the scaled return value at selector plus `0x11`.

After intervening calls, it reads the byte at that source offset again. It writes the low byte of bits 20–29 of the unsigned return value at the freshly selected address plus `0x11`.

After more calls:

1. Read a byte at address `1024 × scaled return value + 1`.
2. Write zero at that selector plus `0x11`.
3. Call a dependency with the scaled return value multiplied by eight twice, the scaled return value itself, and four times another call result.

Following additional calls, the byte read and zero write are repeated using the same computed source address but a fresh selector read.

Further indirect calls occur. The representation then ends at bad instruction data, without an established normal exit.

## Side effects

Explicit memory effects include:

- Byte-selected writes to addresses from `0x11` through `0x110`.
- Direct byte writes to absolute addresses `0x0d` and `0x11`.
- Computed byte fields at offsets 1, 7, and `0x0d`.
- Computed 32-bit fields at offsets 4, `0x14`, `0x24`, `0x34`, and `0x74`.
- Repeated reads and stores separated by calls.

These offsets do not establish a structure layout. Bases may alias, and byte stores may overlap bytes within word fields. No disjointness guarantee is visible.

No allocation, deallocation, reference counting, ownership acquisition, ownership release, synchronization, or rollback operation is identifiable. Indirect callees may perform any of these operations; their behavior is unknown. In particular, retained addresses are not proven live after intervening calls.

Access failure, callee failure, or nonreturn can leave a partially modified state. No explicit recovery path is shown.

## Dependencies

Only F-f6dabb4f53c5 has a supplied stable function identity. All other dependencies are unresolved indirect targets.

For documentation, **F-unknown-entry** denotes the first target, derived from incoming execution state. **F-unknown-indirect** denotes the remaining unresolved target set. These are opaque documentation identities, not claims that the targets are unique, equal, or independently callable.

F-unknown-indirect includes calls used to:

- Obtain 32-bit values.
- Obtain apparent paired 32-bit results.
- Produce secondary register outputs.
- Consume one through four visible arguments.
- Execute with no visible arguments and no subsequently used ordinary return value.

A call with no visible arguments is not proven to have no inputs. No dependency is proven to preserve globals, retain memory validity, release memory, or obey the recovered prototype.

## Acceptance examples

These examples are conditional static expectations, not observed runtime results.

1. **Initial selector write.** If the incoming address-bearing value is `0xab001000` and the byte at its offset 1 is `0x20`, the first explicit byte store targets `0x31` and writes `0xab`, provided preceding calls return.

2. **Selector limits.** A zero selector targets `0x11`; a selector of `0xff` targets `0x110`. The selector is an unsigned byte, not a signed displacement.

3. **Packed extraction.** If the word read at the relevant scaled-base offset 4 is `0xffc00000`, its right shift by 22 is `0x3ff`. The duplicated masked arguments are `0x3fe00000`. Under wrapping 32-bit arithmetic, the later shift by 23 stores `0xff800000`.

4. **Split-result replacement.** If the earlier eightfold index is `0x80`, the two corresponding word stores target `0xb4`. The later store contains the high half of the apparent 64-bit result, regardless of the first stored payload. Effects between those stores remain observable.

5. **Fresh selectors.** If an intervening dependency changes a selector source byte from `0x10` to `0x20`, subsequent selector reads address different destinations, `0x21` and `0x31`. Reusing a cached selector would not preserve the visible access sequence.

6. **Low-address overwrite.** If the small selector is `0x12`, the first direct write to address `0x0d` stores `0x12`. Reaching the later direct write stores `0xc0` there. This does not establish the value after all subsequent callees run.

7. **Failure prefix.** If any required address is inaccessible, later stores and calls are not guaranteed to occur. The evidence supplies no required error code, rollback, or exception category.

## Uncertainties

The following are **documented_only**:

- Whether the recovered instruction sequence accurately represents an executable function.
- The routine’s application-level intent.
- Its complete ABI, including hidden inputs and secondary outputs.
- Concrete dependency identities and contracts.
- Signed-overflow and invalid-shift behavior.
- Endianness, alignment, address validity, and memory-mapped device behavior.
- Ownership, lifetime, aliasing, reentrancy, and concurrency guarantees.
- Whether apparent 64-bit results are genuine scalar returns or combined register values.
- The actual behavior at the undecodable endpoint.

Repeated scaling by four suggests possible word-indexed representations, and selector-plus-offset writes suggest possible compact state storage. Both are inferred intent only, not established types or layouts.

## Evidence

Established static evidence comprises the supplied representation’s declared argument widths, visible indirect calls, explicit memory-access widths and ordering, masks, shifts, arithmetic, repeated low-address writes, apparent paired results, and truncated endpoint.

No runtime tests have been performed. No dependency implementation, calling-convention specification, memory map, or execution trace was supplied. This contract therefore distinguishes visible-prefix obligations from unresolved execution semantics and does not claim verified end-to-end behavior.
