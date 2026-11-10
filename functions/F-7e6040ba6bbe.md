<!-- Trusted provenance: F-7e6040ba6bbe; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-7e6040ba6bbe

## Purpose

F-7e6040ba6bbe specifies a sequence of argument-derived, incoming-state-derived, fixed-global, and dependency-address-derived memory accesses. The represented sequence has an early conditional invalid-instruction termination marker and a final unconditional marker. It exposes neither a normal return path nor a recoverable error result.

No application-level purpose is established. Control-state-related labels suggest low-level exception or reset machinery, but that interpretation is **inferred intent**, not an acceptance requirement.

This contract distinguishes:

- **Established static evidence:** access widths, represented arithmetic, ordering, branches, and termination markers visible in the supplied representation.
- **Inferred intent:** possible interpretations that do not add requirements.
- **documented_only:** unresolved execution semantics, storage layouts, environmental requirements, and runtime consequences.

All acceptance examples are conditional static expectations. No runtime tests have been performed.

## Interface

### Explicit inputs

| Input | Represented width | Role |
|---|---:|---|
| First argument | Signed 32 bits | Arithmetic, byte data, address generation, and index generation |
| Second argument | 8 bits | Data for an early byte write |
| Third argument | Signed 32 bits | Initial byte data and an address base |
| Fourth argument | Signed 32 bits | Byte-write address base and an ambiguously shifted word-address base |

No explicit result value is supplied.

### Implicit inputs

The sequence additionally consumes:

- Two incoming 32-bit register values not initialized in the visible body, designated the **first implicit base** and **second implicit base**.
- A pointer obtained from an incoming stack location at displacement `0x250`, designated the **incoming pointer**.
- The address of that stack location, independently of the pointer stored there.
- Contents of explicitly read memory locations.
- Fixed global storage, including a word at `0x0897ea50`, a word at `0x0897ea80`, an instruction-related overlapping byte, and a reset-related halfword.
- The addresses associated with F-e47d7b2e9c88 and F-8afdaab91693.

The representation uses 32-bit pointer-sized values and address calculations. Actual address mapping, relocation, and pointer validity remain **documented_only**.

### Preconditions and completion

No validation, bounds check, alignment check, ownership check, or permission check is visible. Reaching a later operation requires every earlier operation on that path to complete.

The caller cannot infer that any computed address is mapped, writable, aligned, or owned. Computations can target zero, other low addresses, dependency-associated storage, or storage already accessed earlier.

There is no represented normal completion. Either:

1. The early signed-address condition reaches an invalid-instruction marker; or
2. The remaining sequence reaches the final invalid-instruction marker.

An earlier access or calculation may prevent either marker from being reached. The representation does not establish a signal, exception type, exit status, reset, recovery protocol, or resumability.

## Behavior

### Arithmetic and descriptive quantities

A **byte** is 8 bits, a **halfword** is 16 bits, and a **word** is 32 bits. Explicit narrowing retains the low destination-width bits.

The following quantities describe the contract:

| Quantity | Definition |
|---|---|
| Offset value | First argument minus `0x38`, then held as an unsigned 32-bit value |
| Unsigned difference | First argument converted to unsigned 32 bits, minus `0xe0`, modulo \(2^{32}\) |
| Halfword-region address | Twice the unsigned difference, modulo \(2^{32}\) |
| Masked difference | Unsigned difference with bit 31 cleared |
| Index | Masked difference shifted right by one |
| First dependency address | Address associated with F-e47d7b2e9c88 |
| Second dependency address | Address associated with F-8afdaab91693 |
| Ambiguous word base | Fourth argument shifted left by two, then right by 32 |
| Ambiguous dependency base | First dependency address interpreted as signed, then shifted right by 32 |

The index ranges from `0` through `0x3fffffff`.

The offset subtraction is represented as signed arithmetic before assignment to unsigned storage. Its overflow behavior is not established. Likewise, the shifts by 32 do not have an established ordinary 32-bit meaning. Signed left shifts, signed multiplication, and other signed address arithmetic are unresolved where their represented operations exceed the applicable range.

These qualifications do **not** apply indiscriminately to every calculation. In particular, unsigned-difference arithmetic, halfword-region multiplication, and the indexed signed-halfword read described below have explicit unsigned 32-bit behavior.

### Initial accesses

The first three memory effects occur in this order:

1. Write the low byte of the third argument at the second implicit base plus `3`.
2. Write the low byte of the first implicit base at the third argument plus `0x10`.
3. Write the second argument at the first implicit base plus `0x1c`.

The argument-derived region is then modified in this order:

1. Write the low byte of the index at the fourth argument plus `0x0897e8b8`.
2. Assign `0xb8` to the global at `0x0897e8c0`.
3. Write the low byte of the offset value at the first argument minus `0x32`.
4. Write the offset value as a word at the ambiguous word base.
5. Write halfword `0xffff` at halfword-region byte offset `4`.
6. Write halfword `0xffff` at halfword-region byte offset `6`.
7. Write byte `0xb8` at ambiguous-word-base byte offset `4`.
8. Write halfword `0xffff` at halfword-region byte offset `2`.
9. Assign the low 16 bits of the halfword-region address to the global at `0x0897ea28`.

The supplied representation does not establish the complete storage widths of the globals at `0x0897e8c0` and `0x0897ea28`. The latter assignment explicitly supplies a 16-bit value; this alone does not prove the destination’s complete declared extent.

Next, read an **unsigned halfword** at halfword-region byte offset `0`. Zero-extend that sample and use it as the destination address of a word write. The written word is the offset value masked with `0x897e8b80`.

The sampled halfword is not one of the three explicitly initialized halfwords. Its value can nevertheless depend on earlier operations through aliasing. Its zero-extended address lies between `0` and `0xffff`, inclusive.

### First dependency-address phase

No invocation of F-e47d7b2e9c88 is visible. Its address is used as data, as an address, and in a branch condition.

Interpret the first dependency address as signed, shift it right by 31, and narrow the result to a halfword. Under conventional 32-bit arithmetic-right-shift semantics, this **sign halfword** is:

- `0x0000` for a nonnegative address;
- `0xffff` for a negative address.

Write the sign halfword at ambiguous-dependency-base byte offset `4`. Only after that write is the signed first dependency address tested for negativity.

If negative, reach the early invalid-instruction marker. No subsequent effect is represented on that path, and no rollback is visible.

Otherwise, perform these effects in order:

1. Write the sign halfword at ambiguous-dependency-base byte offset `0x12`.
2. Write the first dependency address as a word to the address it denotes.
3. Read a signed halfword from the indexed address defined below.
4. Repeat the halfword write at byte offset `0x12`.
5. Repeat the dependency-address self-write.
6. Sign-extend the sampled halfword to 32 bits, shift right by 16, and write the resulting word at ambiguous-dependency-base byte offset `8`.

The indexed read uses the index as a **byte displacement**, not a halfword index. Because the index is unsigned 32-bit, the ambiguous dependency base is converted to unsigned 32 bits for this addition. The sampled address is therefore:

\[
(\operatorname{unsigned32}(\text{ambiguous dependency base})+\text{index})
\bmod 2^{32}.
\]

This addition has established modulo-\(2^{32}\) behavior even though the preceding base-producing shift by 32 remains unresolved. Address validity and alignment remain unresolved as well.

Under conventional arithmetic-right-shift semantics, the final word in this phase is zero for a nonnegative signed-halfword sample and `0xffffffff` for a negative sample.

The repeated writes are distinct effects. Their repetition must not be discarded where aliasing or device semantics could make it observable.

### Low-address control writes

Assign zero to an abort-related global.

Define the **anchor** as `0x58` plus the value obtained by retaining only bit 2 of the first dependency address. The anchor is therefore either `0x58` or `0x5c`.

Continue in this order:

1. Write the ambiguous dependency base as a word at anchor plus `4`.
2. Assign the masked difference shifted right by 17, narrowed to a halfword, to a low-address control global.
3. Assign the low byte of the index, widened to a halfword, to an instruction-related global.
4. Write the index as a word at four times the anchor plus `0x10`.
5. Assign the index to a supervisor-call-related global.

The value supplied in step 2 ranges from `0` through `0x3fff`. Step 3 supplies a value between `0` and `0xff`. Complete destination storage extents are not established for these global assignments.

### Incoming-pointer phase

The incoming pointer participates in the following ordered effects:

1. Write the anchor as a halfword at its base.
2. Overwrite its base with a word containing the ambiguous dependency base shifted left by eight.
3. Write that same shifted word at incoming-pointer byte offset `20`.
4. Overwrite its base with a word containing the incoming pointer’s numeric address.
5. Read a word at twice the incoming pointer’s numeric address.
6. Write the low halfword of that sample at the incoming pointer’s base.
7. Write the complete sample as a word to the address denoted by the sample itself.

The final operation treats sampled memory contents as an address. It is not merely another write within the incoming pointed-to object.

Next, interpret the incoming pointer’s numeric value as signed and shift it right by 31. Write that word at a destination formed from the **address of the incoming stack slot**, multiplied by `0x20`, plus `0x18`. This destination is not calculated from the incoming pointer’s pointee address.

Then read the current instruction-related overlapping byte and assign the instruction-related composite formed by concatenating:

- A 16-bit value of `5` as the upper component;
- The sampled 8-bit value as the lower component.

The resulting 24-bit value is `0x000500` combined with the sampled byte. Bits 8–15 contain `0x05`; bits 16–23 are zero. Bits 0–7 contain the sampled byte. The destination’s complete storage width remains unresolved.

This byte is read at this point in the sequence. Its value is not guaranteed to equal the low byte of the index assigned earlier, because intervening effects may alias it.

Finally, write a word containing the incoming pointer’s numeric address at four times that address plus `0x18`.

### Final global-derived phase

Read the word at `0x0897ea50` now and retain it as the **sampled global word**. Earlier effects may have changed that location. The subsequent calculations use this sample, not an assumed entry-time value.

Assign four times the incoming pointer’s numeric address to the word global at `0x0897ea80`. Then:

1. Obtain both a destination address and stored word from read expressions referring to `0x0897ea80`, and perform that indirect word write.
2. After that write completes, read `0x0897ea80` again, add `4`, and write the index as a word at the resulting address.

The representation does not establish the relative evaluation order of the destination and payload reads within step 1, or their physical load count. It does establish the later global-read expression in step 2. Replacing these reads with the originally assigned scaled pointer requires stable storage and noninterference.

Continue with:

- A word write at four times the sampled global word, containing that same scaled value.
- A word write containing the second dependency address shifted left by two, at the sampled global word plus the result of shifting that sample left by six and then right by 17.

Signed-operation qualifications apply to these represented calculations.

Assign the sampled global word’s upper 16 bits, obtained by an unsigned right shift, to the reset-related halfword slot. Then assign zero to an interrupt-related global.

At a base formed by eight times the second dependency address:

1. Write a zero word at byte offset `0`.
2. Write a zero word at byte offset `4`.
3. Write a byte at offset `16` equal to the low byte of that base plus eight, reduced to byte width.

Next perform these fixed-global assignments in order:

| Destination | Supplied value |
|---|---:|
| `0x00000086` | `0x897ea700` |
| `0xea700000` | `0x66` |
| `0x0897eb26` | `0xeaec` |
| `0x12fd4c34` | `0x1a` |
| `0x897ea65c` | Its own address |

Their complete physical write extents are not all established by the supplied representation.

After these assignments, **read the reset-related halfword again**. Form the reset composite with that current sample as its upper 16 bits and `0xea70` as its lower 16 bits, then assign the composite to the reset-related global.

This is not guaranteed to reuse the previously assigned halfword value. Intervening writes may affect it through unresolved overlap or indirect aliasing.

Finally, assign `0x5c` to the global at `0x12fd4cb8` and reach the unconditional invalid-instruction marker.

## Side effects

Observable effects include explicit byte, halfword, and word accesses; fixed-global assignments; low-address writes; dependency-address self-writes; and writes through addresses obtained from sampled data.

No allocation, deallocation, ownership transfer, synchronization operation, or dependency invocation is visible. This absence does not establish ownership, lifetime, or safe access to any affected storage.

No transactional boundary or rollback is represented. An access failure or termination can leave a completed prefix of effects visible.

Aliasing matters throughout. In particular:

- Earlier writes may change the unsigned halfword sampled from the argument-derived region.
- Repeated dependency-address writes remain separate effects.
- The instruction composite uses a current overlapping-byte read.
- The scaled-pointer global is read again after its first indirect store.
- The reset composite uses a late halfword read after intervening writes.

Endianness, access atomicity, alignment requirements, permissions, concurrency, and device-memory behavior remain **documented_only**.

## Dependencies

| Opaque ID | Established use | Not established |
|---|---|---|
| F-e47d7b2e9c88 | Address-based sign extraction, conditional termination, address arithmetic, self-writes, indexed read base, and anchor selection | Invocation, return value, executable behavior, ownership, or preservation of globals |
| F-8afdaab91693 | Address-based shifted payload and multiplied write destinations | Invocation, return value, executable behavior, ownership, or preservation of globals |

Neither dependency is visibly called. Their executable behavior contributes no inferred memory-management or global-preservation guarantees.

## Acceptance examples

These examples require preceding operations to complete and any stated arithmetic or noninterference assumptions to hold. They are not executed tests.

1. **Zero unsigned difference.** A first-argument bit pattern of `0x000000e0` produces unsigned difference zero, halfword-region address zero, masked difference zero, and index zero. This does not establish safe execution.

2. **Unsigned subtraction boundary.** A first-argument bit pattern of `0x000000df` produces unsigned difference `0xffffffff`, halfword-region address `0xfffffffe`, masked difference `0x7fffffff`, and index `0x3fffffff`. The early index-byte write supplies `0xff`.

3. **Unsigned region sample.** If the halfword at region offset `0` is `0x1234` when read, the following masked-word write targets `0x00001234`. Initializing offsets `2`, `4`, and `6` does not by itself establish that sample.

4. **Early termination ordering.** If the first dependency address is signed-negative, its sign-derived halfword write at ambiguous-base offset `4` precedes the early termination marker. The later dependency self-writes are not reached.

5. **Unsigned indexed-address wrap.** Conditional on an ambiguous dependency base whose unsigned value is `0xfffffff0` and an index of `0x20`, the signed-halfword read targets `0x00000010`. This addition wraps modulo \(2^{32}\); it does not establish how the ambiguous base was produced or whether the resulting address is accessible.

6. **Signed-halfword propagation.** Under conventional arithmetic-right-shift semantics, a sampled halfword of `0x8000` produces word `0xffffffff` at ambiguous-base offset `8`. A sample of `0x7fff` produces zero.

7. **Instruction composite placement.** If the instruction-related byte is `0xa7` when sampled, the composite expression is `0x0005a7`, not `0x0500a7`. Its physical write extent remains unresolved.

8. **Sampled data as destination.** If the word read from twice the incoming pointer is `0x00002000`, the subsequent full-word self-write targets `0x00002000` and supplies `0x00002000`.

9. **Stable scaled-pointer global.** If the assignment to `0x0897ea80` produces `0x00002000`, all relevant reads remain stable, and the first indirect store does not affect that global, the stores target `0x00002000` and `0x00002004`. Without those conditions, the second destination follows the later global read.

10. **Late reset read.** If the earlier reset-halfword assignment supplies `0x1234` but an intervening effect changes the slot to `0xabcd` before the final read, the reset composite is `0xabcdea70`. With noninterference instead, it is `0x1234ea70`.

11. **No normal return.** If the early branch continues and every remaining operation completes, the final outcome represented is the unconditional invalid-instruction marker, not a normal return.

## Uncertainties

The following remain **documented_only**:

- Whether the supplied representation corresponds to executable instructions rather than misinterpreted data.
- Actual execution semantics of both shifts by 32.
- Signed overflow, signed left shifts, and signed address multiplication where applicable.
- Full calling convention and validity of incoming register and stack state.
- Complete global widths, overlap layouts, and relocation.
- Physical load count and within-expression ordering for the first `0x0897ea80`-derived store.
- Effects of aliasing, concurrency, or device-driven mutation on later reads.
- Endianness, alignment, atomicity, permissions, and privilege requirements.
- Runtime consequences of the termination markers or preceding access failures.
- Any application-level purpose.

The unsigned indexed-address addition is not an unresolved signed-overflow case. Its modulo-\(2^{32}\) behavior is established; the base-producing shift and access validity are not.

## Evidence

This contract relies only on the supplied static representation and opaque dependency mapping. No implementation access, instruction-level validation, runtime execution, or tests were performed.

The representation supports the stated ordered accesses, explicit widths, unsigned arithmetic, dependency-address uses, conditional branch, and termination markers. It also reports invalid instruction data and overlapping global symbols, limiting stronger executable or layout claims.

The instruction-related composite combines a 16-bit upper component with an 8-bit lower component, placing the value `5` at bits 8–15. The signed-halfword sample address combines a signed base with an unsigned 32-bit index, establishing unsigned 32-bit addition.

Current global-read expressions support the late-read requirements for both the scaled-pointer global and the reset halfword. Treating those reads as unchanged earlier values requires assumptions not established by the supplied material.
