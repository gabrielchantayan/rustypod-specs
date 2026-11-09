<!-- Trusted provenance: F-be1cf53b6a68; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-be1cf53b6a68

## Purpose

The supplied static representation describes a routine that interleaves 29 indirect dependency calls with three 32-bit stores, two byte reads, and two byte stores. Its observable behavior depends on an unsigned input word, implicit entry-register state, selected dependency results, and memory contents.

No application-level purpose is established. The input is used in address arithmetic and bit extraction, but the evidence does not identify it as an object pointer, resource handle, or encoded command. Likewise, the dependency calls are not established as allocation, initialization, serialization, or device operations.

The representation reports invalid instruction data and truncated control flow. This contract describes the represented operations and their ordering, conditional on that representation being accurate. It does not establish that an executable invocation reaches all operations or returns normally.

## Interface

### Explicit input

The routine accepts one unsigned 32-bit value, called the **input word** below.

The input participates in:

- The address of the first 32-bit store.
- Two explicit arguments to a dependency.
- The address used by both byte reads.

No input validation is represented.

### Implicit inputs and preconditions

The routine also consumes:

- A 32-bit value present in register R7 at entry.
- The entry link-register value, which contributes to the first indirect-call target.
- Results returned by selected dependencies.
- Memory bytes obtained through two separate reads.
- Executable targets for the represented indirect calls.

For an execution to progress through the full represented sequence, every reached memory access must be permitted, every indirect target must be executable under the actual calling convention, and every dependency must return. These are conditional execution requirements, not checks performed by the routine.

No containing allocation, memory extent, alignment guarantee, or ownership relationship is established for any computed address.

### Outputs

The represented interface has no return value. Observable outputs are direct memory effects and dependency invocations, together with any effects of those dependencies.

The representation ends at an invalid-instruction termination marker rather than a normal return. It establishes neither a success status nor an error result.

### Widths and arithmetic

The input word, retained scalar results, and full-width stores are represented as 32 bits. Byte reads and byte stores are 8 bits.

For unsigned input arithmetic and unsigned shifts, this contract uses 32-bit arithmetic: results retain the low 32 bits. In particular, input-plus-20 and the byte-lookup address calculation wrap modulo \(2^{32}\).

A byte read produces a value from 0 through 255. Adding 17 therefore yields an address from 17 through 272 without overflow. A byte store retains only the source value’s low eight bits.

Several dependency results are represented as signed 32-bit values before multiplication or left shifting. Their formulas below express the represented arithmetic, but do not establish portable behavior for signed overflow or negative left operands. The unsigned bit patterns used for the byte stores remain distinguishable from these unresolved signed-arithmetic cases.

## Behavior

### Dependency naming and ordering

The dependency identifiers are opaque labels for call sites. They do not imply known implementations, distinct targets, or semantic roles beyond the observed argument and result uses.

A call described as having “no explicit arguments” has no arguments shown in the supplied representation. That does not establish a zero-argument machine-level interface.

Subject to faults, non-returning dependencies, and the representation’s control-flow limitations, the following phases occur in order.

### 1. Entry store and first arithmetic result

Before any dependency invocation, the routine writes the entry R7 word as a 32-bit value at the address formed by adding 20 to the input word using unsigned 32-bit arithmetic.

It then invokes:

1. **F-d00000000001**, with the input word.
2. **F-d00000000002**, with no explicit arguments.
3. **F-d00000000003**, with no explicit arguments.
4. **F-d00000000004**, with no explicit arguments.
5. **F-d00000000005**, with no explicit arguments.

The first target is derived from the entry link-register value plus 4,094 bytes.

The result of **F-d00000000005** is the **first arithmetic result**. The routine derives a **scaled word** by multiplying that result by four, then performs a 32-bit store:

- Address: sixteen times the first arithmetic result, plus 20.
- Stored value: the scaled word.

Both multiplications involve a result represented as signed. Overflow behavior is not established by the supplied evidence.

The next call is **F-d00000000006**, with three explicitly represented 32-bit arguments:

1. The input’s low six bits shifted left by 14.
2. The scaled word.
3. The input word shifted left by 28, retaining the low 32 bits.

The first argument ranges from 0 through 1,032,192 and occupies bits 14–19. The third argument depends only on the input’s low four bits.

### 2. Second computed-address store

The routine invokes, in order:

1. **F-d00000000007**
2. **F-d00000000008**
3. **F-d00000000009**
4. **F-d0000000000a**

No explicit arguments are shown.

The result of **F-d0000000000a** is the **second arithmetic result**. The routine writes the previously derived scaled word as a 32-bit value at four times the second arithmetic result, plus 20.

It then invokes **F-d0000000000b** with four times the second arithmetic result.

The first and second computed-address stores use the same represented stored value. Their address expressions differ, but their destinations are not guaranteed to differ. Signed multiplication and address validity remain conditional.

### 3. First byte lookup and first byte store

The routine next invokes:

1. **F-d0000000000c**
2. **F-d0000000000d**
3. **F-d0000000000e**
4. **F-d0000000000f**

No explicit arguments are shown. The result of **F-d0000000000f** is retained as an unsigned 32-bit **bit-field result**.

The **lookup address** is the input word multiplied by 67,108,864, followed by addition of one, using unsigned 32-bit arithmetic. Equivalently, it is the input’s low six bits shifted left by 26, plus one.

The possible lookup addresses are exactly:

\[
1 + k \times 67{,}108{,}864,\qquad 0 \leq k \leq 63.
\]

The routine reads an unsigned byte from this address. It then writes the low eight bits of the bit-field result shifted left by two to the address equal to the loaded byte plus 17.

The loaded byte supplies the destination’s numeric base. This is not a store at the lookup address plus 17.

Consequently:

- The byte-store destination is between 17 and 272 inclusive.
- The stored byte is formed from bits 0–5 of the bit-field result, shifted left by two.
- The stored byte’s low two bits are zero.

After this store, **F-d00000000010** is invoked with three 32-bit arguments:

1. The bit-field result shifted left by two.
2. The same shifted value.
3. The bit-field result shifted left by four.

These shifts originate from an unsigned result and retain the low 32 bits. One shifted value is represented through a signed temporary; its signed interpretation is not needed to specify the low-byte store.

### 4. Intervening calls and second byte lookup

The routine invokes **F-d00000000011** with no explicit arguments. Its represented 32-bit result is passed directly to **F-d00000000012**.

The routine then invokes:

1. **F-d00000000013**
2. **F-d00000000014**
3. **F-d00000000015**
4. **F-d00000000016**

No explicit arguments are shown. The result of **F-d00000000016** is the **third arithmetic result**.

The routine now performs a second unsigned-byte read from the same lookup address. This is a separate observable read, not reuse of the first loaded value.

Using the newly loaded byte, it performs the second byte store:

- Address: the second loaded byte plus 17.
- Stored value: bits 20–27 of the bit-field result.

The represented value calculation first masks the bit-field result to its low 30 bits and shifts right by 20. That intermediate can contain ten bits, but the byte store retains only eight. Bits 28–31 therefore do not affect the stored byte.

The second loaded byte is retained, zero-extended to 32 bits, for a later dependency argument.

### 5. Final call sequence and represented termination

After the second byte store, the routine invokes:

1. **F-d00000000017**, with the third arithmetic result shifted left by two.
2. **F-d00000000018**, with no explicit arguments.
3. **F-d00000000019**, with no explicit arguments.
4. **F-d0000000001a**, with no explicit arguments.
5. **F-d0000000001b**, with no explicit arguments.

The result of **F-d0000000001b** is the **fourth arithmetic result**.

The routine then invokes **F-d0000000001c** with:

1. The fourth arithmetic result shifted left by two.
2. The saved byte from the second lookup, zero-extended to 32 bits.

The shifts of the third and fourth arithmetic results involve signed represented values. Negative operands and overflow remain unresolved.

Finally, **F-d0000000001d** is invoked with no explicit arguments. The representation then terminates at an invalid-instruction marker. No subsequent continuation or normal return is established.

## Side effects

### Direct memory-access order

Excluding dependency-internal accesses, the represented order is:

| Order | Direct access | Width |
|---|---|---:|
| 1 | Store entry R7 word at input plus 20 | 32 bits |
| 2 | Store scaled word at sixteen times the first arithmetic result plus 20 | 32 bits |
| 3 | Store scaled word at four times the second arithmetic result plus 20 | 32 bits |
| 4 | Read first byte from the lookup address | 8 bits |
| 5 | Store shifted low-bit byte at first loaded byte plus 17 | 8 bits |
| 6 | Read second byte from the lookup address | 8 bits |
| 7 | Store bits 20–27 at second loaded byte plus 17 | 8 bits |

Dependency calls intervene as specified in the behavior section. Their effects cannot be omitted when reasoning about memory contents between direct accesses.

### Aliasing and intervening mutation

The lookup-address set and the byte-store destination set are disjoint for every input under the stated unsigned 32-bit numeric address calculation.

Every lookup address is congruent to 1 modulo 67,108,864. The only such address below 273 is 1, whereas every byte-store destination is between 17 and 272 inclusive. Therefore, neither direct byte store targets the lookup address. In particular, the first byte store cannot directly change the byte subsequently read by the second lookup.

This does not establish that the two reads return equal values. Intervening dependencies may modify the lookup location. Their memory effects are unknown.

Other aliasing remains unresolved:

- The two byte-store destinations are equal exactly when the two loaded bytes are equal.
- No disjointness is established for the three 32-bit stores relative to one another or to other accessed storage.
- Dependencies may read or modify direct-store destinations.
- Dependencies between the two byte stores may change a shared destination before the later direct write.

Equal byte-store destinations therefore establish two writes to the same address at different points, not preservation of the first value until the second write.

### Ownership and state

No explicit allocation, release, ownership transfer, or resource acquisition is identified.

Passing a word to a dependency does not establish borrowing, consumption, retention, or release. Dependency ownership and lifetime effects remain unknown.

Selected values are represented as retained across calls. This does not imply that dependencies preserve arbitrary globals, memory, registers, or external state.

### Errors and interruption

No explicit validation, status check, exception handler, recovery branch, or rollback is represented.

If a memory access faults or a dependency fails to return, subsequent represented operations need not occur. Earlier effects are not visibly undone. Actual fault handling and the runtime meaning of the terminal invalid-instruction marker are unknown.

## Dependencies

| Opaque identifiers | Established call-site role |
|---|---|
| **F-d00000000001** | Receives the input; target depends on entry link-register state. |
| **F-d00000000002–F-d00000000004** | Precede production of the first arithmetic result. |
| **F-d00000000005** | Produces the first arithmetic result. |
| **F-d00000000006** | Receives two input-derived patterns and the scaled word. |
| **F-d00000000007–F-d00000000009** | Precede production of the second arithmetic result. |
| **F-d0000000000a** | Produces the second arithmetic result. |
| **F-d0000000000b** | Receives four times that result. |
| **F-d0000000000c–F-d0000000000e** | Precede production of the bit-field result. |
| **F-d0000000000f** | Produces the unsigned bit-field result. |
| **F-d00000000010** | Receives three shifted bit-field arguments. |
| **F-d00000000011–F-d00000000012** | Produce and immediately consume a represented 32-bit result. |
| **F-d00000000013–F-d00000000015** | Precede production of the third arithmetic result. |
| **F-d00000000016** | Produces the third arithmetic result. |
| **F-d00000000017** | Receives that result shifted left by two. |
| **F-d00000000018–F-d0000000001a** | Precede production of the fourth arithmetic result. |
| **F-d0000000001b** | Produces the fourth arithmetic result. |
| **F-d0000000001c** | Receives its shifted value and the saved second lookup byte. |
| **F-d0000000001d** | Last represented dependency invocation. |

Most targets use a fixed or symbol-derived address plus 4,094 bytes. **F-d00000000010** instead uses a displacement of 4,076 bytes. These constructions establish neither executable validity nor callee semantics.

## Acceptance examples

These are conditional arithmetic examples, not executed tests. Each assumes that the relevant operations are reached, accesses are valid, dependencies return, and the static representation is accurate.

### Input bit selection

For input 1:

- The first 32-bit store targets address 21.
- **F-d00000000006** receives first argument `0x00004000` and third argument `0x10000000`.
- Both byte reads target `0x04000001`.

The middle argument depends on **F-d00000000005**.

### Unsigned boundary input

For input `0xFFFFFFFF`:

- The first store address wraps to 19.
- The lookup address is `0xFC000001`.
- **F-d00000000006** receives first argument `0x000FC000` and third argument `0xF0000000`.

Regardless of the bytes read, both byte-store destinations remain within 17–272 and cannot equal that lookup address.

### Scaled stores without signed overflow

If **F-d00000000005** returns 3 and **F-d0000000000a** returns 5:

- The scaled word is 12.
- The first computed-address store writes 12 at address 68.
- The second writes 12 at address 40.
- **F-d0000000000b** receives 20.

These values avoid the unresolved signed-overflow cases.

### Different lookup results

If the bit-field result is `0x3AB12345`, the first lookup returns 7, and the second returns 9:

- The first byte store writes `0x14` at address 24.
- The second byte store writes `0xAB` at address 26.
- **F-d0000000001c** receives 9 as its second explicit argument.

The intervening dependencies are permitted to change the lookup location. Reusing the first loaded byte instead of performing the second read would not satisfy the represented behavior.

### Same byte destination with intervening mutation

If both lookups return 7 and the bit-field result is `0x3AB12345`, the routine directly writes `0x14` to address 24 earlier and `0xAB` to address 24 later.

Nothing establishes that address 24 still contains `0x14` immediately before the later write. Intervening dependencies may have modified it. Likewise, later dependency effects may change it after the `0xAB` write.

### Interrupted execution

If the first direct store faults, no represented dependency invocation precedes that fault.

If a later dependency does not return, only the execution prefix through that invocation is established. No rollback is represented.

## Uncertainties

### Established static evidence

The supplied representation shows the direct-access order, dependency-call order, explicit argument expressions, selected result uses, two separate byte reads, and terminal invalid-instruction marker.

Unsigned arithmetic establishes the lookup-address set and its disjointness from all direct byte-store destinations. That conclusion does not depend on dependency memory preservation.

### Inferred intent

The bit extraction and repeated calls suggest manipulation of encoded values, but no application meaning is established. The invalid-instruction warnings may reflect unusual control flow or an inaccurate static representation; neither explanation is proven.

### documented_only

The following remain documented_only:

- Actual instruction validity and reachable control flow.
- Dependency identities, signatures, calling convention, and omitted arguments.
- Dependency memory mutations, global-state changes, failures, and non-returning behavior.
- Dependency ownership and resource-lifetime effects.
- Address validity, alignment, byte order, and memory-mapped behavior.
- Signed overflow, negative left shifts, and out-of-range signed interpretation.
- Actual fault handling and invalid-instruction handling.
- Any continuation or normal return absent from the representation.

## Evidence

This contract is based solely on the supplied static function representation and its warnings. The prior draft and reviewer feedback are not independent execution evidence.

The address-set disjointness follows from unsigned 32-bit arithmetic. The same-destination examples establish ordered direct writes without assuming that intervening dependencies preserve memory.

No runtime tests have been performed. No dependency bodies, executable observations, or successful execution results are available.
