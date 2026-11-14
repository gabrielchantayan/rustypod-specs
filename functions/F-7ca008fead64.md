<!-- Trusted provenance: F-7ca008fead64; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-7ca008fead64

## Purpose

F-7ca008fead64 is represented as a non-returning sequence of indirect dependency calls, computed-address word stores, byte-addressed reads and writes, and an unconditional repeating tail.

The supplied static representation does not establish an application-level purpose. In particular, it does not justify describing the function as a serializer, allocator, device driver, parser, or cleanup routine. Its observable contract is the ordering and data dependence of its calls and memory accesses.

The representation carries warnings about invalid instruction data and omitted unreachable regions. Accordingly, this document distinguishes:

- **Established static evidence:** operations and ordering explicitly present in the supplied representation.
- **Inferred interpretation:** 32-bit machine arithmetic used to express computed values and addresses.
- **Unknown behavior:** dependency effects, actual calling convention, executable validity, and behavior outside the represented path.

All behavioral requirements below are conditional on the represented path being executable and on each preceding memory access and dependency invocation completing normally.

## Interface

### Explicit inputs

The recovered interface has three argument positions:

| Position | Recovered width and interpretation | Visible use |
|---|---|---|
| First | 32-bit integer, denoted **A** | Stored directly, scaled for addresses, and passed to the first dependency. |
| Second | Opaque 32-bit value, denoted **B** | No explicit use appears in the represented body. |
| Third | Unsigned 32-bit integer, denoted **C** | Passed unchanged to the first dependency and later passed after a logical right shift by 14 bits. |

The absence of an explicit use of B does not establish that its incoming register or stack location is unobservable to dependencies.

### Additional entry state

Two values outside the explicit argument list affect behavior:

- **H:** an incoming 32-bit register value whose initialization is not established. It supplies scaled addresses, argument values, and shifted quantities.
- **L:** an incoming link/control-register value used to compute the first indirect call target by adding `0xEE4`.

A later dependency also supplies an auxiliary register result, denoted **E**. This value is not an ordinary recovered return value, but it directly controls subsequent word stores and a dependency argument.

An implementation cannot reproduce the represented behavior from A, B, and C alone without resolving this additional state.

### Return and termination

The recovered declaration has no return value. The represented control flow has no normal return: after the finite prefix, it enters an unconditional loop with no visible exit condition.

A dependency may fail, trap, fail to return, or transfer control elsewhere. A memory access may fault. None of these possibilities is handled explicitly.

### Arithmetic and layout notation

Word stores are 32 bits; byte reads and stores are 8 bits. Offsets and addresses below are byte quantities.

For compactness:

- **word store at P** means writing four bytes at address P.
- **selector write through S** means reading an unsigned byte at address S, adding `0x11` to that byte value, and writing one byte at the resulting address.
- **four-call bridge** means four consecutive dependency invocations with no explicit arguments in the recovered representation. Only the fourth ordinary return value is selected for subsequent computations.
- **exchange pair** means a no-explicit-argument dependency invocation followed immediately by a dependency invocation receiving its returned 32-bit value.

Address arithmetic, scaling, and left shifts are described as 32-bit bit-vector operations, retaining the low 32 bits. Right shifts of unsigned quantities are logical. This is a machine-level interpretation, not a claim that signed overflowing expressions in a source language would be defined.

No endianness is established. A word store therefore specifies width and value, not the order of its constituent bytes.

## Behavior

### Dependency reference convention

All unresolved dependency invocations are referenced collectively as **F-000000000000**. This is an opaque documentation identifier for the unresolved target set, not a claim that every invocation reaches the same function.

The sequences below preserve the explicit arguments recovered at each call site. “No explicit arguments” does not imply that the callee cannot observe registers, globals, stack contents, or prior memory writes.

### 1. Entry publication and first result-derived stores

Before the first dependency invocation, the function stores A as a word at:

**4A + `0x14`.**

It then invokes F-000000000000 at the target derived from L plus `0xEE4`, with four explicit arguments:

**4A, A, C, and 8H.**

The subsequent sequence consists of:

1. An exchange pair.
2. A second exchange pair whose consuming invocation also receives **H logically shifted right by 8**.
3. A four-call bridge, producing **R₁**.

The function then stores A again at **4A + `0x14`**. This is a distinct second store, separated from the first by dependency activity.

It invokes a dependency with **4R₁**, then runs a four-call bridge producing **R₂**. It stores **4R₂** at **16R₂ + `0x14`**.

No range check or validity check precedes these computed-address stores.

### 2. H-based publication

The next dependency receives **4R₂** and **8H**. This is followed by:

- An exchange pair.
- Another exchange pair whose consumer additionally receives **H logically shifted right by 8**.
- A four-call bridge producing **R₃**.

The function stores **4R₂** at **8H + `0x14`**.

It then invokes a dependency with **4R₃**, runs a four-call bridge producing **R₄**, and stores **4R₄** at **16R₄ + `0x14`**.

The next dependency receives four explicit arguments:

**8H, 8H, H logically shifted right by 8, and C logically shifted right by 14.**

The final argument results from two successive unsigned seven-bit right shifts. There is no visible use of the intermediate shifted C value as an address.

A four-call bridge produces **R₅**. The function then stores **8H** at **8H + `0x14`**, replacing the earlier value at that same address if no intervening effect changed the memory mapping.

It invokes a dependency with **4R₅**, followed by another four-call bridge. The auxiliary register result E associated with the final invocation becomes the next source value.

### 3. Auxiliary-result publication

The function stores **E multiplied by 2²⁴** at:

**E multiplied by 2²⁶, plus `0x14`.**

Under 32-bit arithmetic, the stored value depends only on E’s low eight bits; the scaled address contribution depends only on E’s low six bits.

A dependency receives **E multiplied by 2²⁴**. A four-call bridge then produces **R₆**.

The function stores that same scaled E value at **4R₆ + `0x14`**, invokes a dependency with **4R₆**, and runs a four-call bridge producing **R₇**.

It next stores **4R₇** at **16R₇ + `0x14`**.

The lifetime and calling-convention validity of E are unresolved. Its presence cannot safely be replaced by an ordinary return value or by an assumed constant.

### 4. Further result publication and selector preparation

A dependency receives **4R₇**. Two exchange pairs follow; neither consuming invocation has an additional explicit argument. A four-call bridge produces **R₈**.

The function stores **4R₇** at **4R₈ + `0x14`**, invokes a dependency with **4R₈**, and runs a four-call bridge producing **R₉**.

Define **P** as the low 32 bits of **4R₉**. The function stores P at **16R₉ + `0x14`**.

A dependency receives P. Two more exchange pairs follow, then a four-call bridge produces **R₁₀**.

The function stores **H logically shifted right by 16** at **4R₁₀ + `0x14`**. It invokes a dependency with **4R₁₀**, runs a four-call bridge producing **R₁₁**, and stores **4R₁₁** at **16R₁₁ + `0x14`**.

The selector base **S** is:

**P with bit 31 cleared, logically shifted right by 7.**

Consequently, S is between zero and `0x00FFFFFF`, inclusive.

A dependency receives **16R₁₁, 16R₁₁, and H logically shifted right by 15**. A four-call bridge produces **R₁₂**.

The function then stores **16R₁₁** at **16R₁₁ + `0x14`**. This is a separate later write to the location that previously received **4R₁₁**.

A dependency receives **4R₁₂**, and a four-call bridge produces **T**.

### 5. Selector-directed byte output

The first byte-output operation reads the selector byte at **S + 1**. It writes the low eight bits of **4T** at the address obtained by adding `0x11` to that selector byte.

The resulting destination is between `0x11` and `0x110`, inclusive. The addition is not an eight-bit wrapping addition.

A dependency then receives **4T** three times as three explicit arguments. An exchange pair follows, then a four-call bridge produces **R₁₃**.

The function rereads the selector byte at **S + 1** and writes the low eight bits of:

**T with its upper two bits cleared, logically shifted right by 20.**

The byte actually stored corresponds to bits 20 through 27 of T. Bits 28 and 29 survive the shift but are discarded by the byte store.

The selector is not cached. Dependencies between these two selector reads may change the selected destination.

A dependency receives **4R₁₃**, and a four-call bridge produces **U**.

### 6. Direct byte output and transition into repetition

The function performs the following two memory operations in order, with no intervening dependency call:

1. Store the low eight bits of **4U** at **H logically shifted right by 23, plus `0x1C`**.
2. Read the selector byte at **S + 1** and write zero through that selector.

These accesses can alias. In particular, the first store may change the selector byte that the second operation reads.

A dependency then receives **H logically shifted right by 23**. One no-explicit-argument invocation follows. Another dependency receives:

**U with its upper two bits cleared, logically shifted right by 6.**

A four-call bridge produces **R₁₄**. The function rereads the selector at **S + 1** and writes zero through it again.

A dependency receives **4R₁₄**, and a four-call bridge produces **V**.

Define:

- **Q** as **V with its upper two bits cleared, logically shifted right by 25**.
- **J** as the low 32 bits of **U shifted left by 12, plus 1**.

Q is between zero and 31, inclusive.

The function reads an unsigned byte at J and writes zero at that byte value plus `0x11`.

It then invokes a dependency with **2Q**, followed by a no-explicit-argument invocation whose ordinary return value, **N**, initializes the repeating tail.

### 7. Unconditional repeating tail

Every iteration has the same externally visible structure:

- A dependency receives N and **Q logically shifted right by 8**.
- A four-call bridge produces **R₁₅**.
- The function reads the selector byte at J and writes zero at the selected address.
- A dependency receives **4R₁₅**.
- A four-call bridge produces **Z**.
- The next iteration uses **4Z** as N and **Z shifted left by 26** as Q.

There is no represented exit test.

On the first iteration, the second explicit argument is necessarily zero because the initial Q is at most 31. On later iterations, that argument is determined by the preceding Z’s low six bits, shifted left by 18. It is not generally zero.

J remains the same computed selector-read address throughout this tail. The selector byte at J is nevertheless reread every iteration, so the selected write destination may change.

## Side effects

### Memory effects

The body explicitly performs word stores and byte stores. Its explicit data-memory reads are selector-byte reads.

The word-store addresses are computed directly from inputs, incoming state, auxiliary results, or dependency returns. No allocation boundary, object extent, null check, alignment check, or access-permission check is visible.

Under the 32-bit arithmetic interpretation:

- Every word-store address is divisible by four.
- Selector-selected byte destinations lie between `0x11` and `0x110`.
- The direct byte destination derived from H lies between `0x1C` and `0x21B`.
- Address arithmetic can wrap.
- Distinct formulas can identify the same address.

These low destination addresses are evidence of unusual addressing, not evidence that the destinations are valid RAM, device registers, or deliberate faults.

Dependencies may observe intermediate stores. Repeated writes therefore cannot be collapsed merely because their addresses or values coincide.

### Ownership and preservation

No ownership transfer, allocation, release, reference-count operation, or object lifetime guarantee is established.

Passing an address-like value to F-000000000000 does not establish whether that dependency reads, retains, modifies, or releases anything. Likewise, no dependency may be assumed to preserve globals or selector memory.

The representation carries derived values across calls. Whether the actual ABI preserves the underlying state required to do so remains unresolved.

### Errors

There is no visible status result, error translation, retry policy, rollback, or compensation.

If an access or dependency invocation fails, the representation does not promise restoration of earlier memory contents. Earlier observable effects may remain.

## Dependencies

**F-000000000000 — unresolved indirect dependency set**

The represented calls include:

- One entry-state-relative target.
- Numerous targets derived from fixed numeric expressions.
- Calls with zero through four explicit arguments.
- Calls whose ordinary return values supply later values or addresses.
- One call whose auxiliary register result supplies later values and addresses.

The static representation does not establish complete prototypes, target identity, target distinctness, ownership behavior, memory effects, exception behavior, or termination.

The four-call bridges and exchange pairs are documentation groupings only. They do not imply that grouped calls are interchangeable, share an implementation, or can be reordered.

## Acceptance examples

These examples specify conditional observations. They are not runtime test results.

1. **Repeated entry store remains observable.**  
   With A equal to 3, the first word store writes 3 at address `0x20`. After the initial call sequence, another word store writes 3 at that address. A dependency changing that location between the stores does not eliminate the second write.

2. **Third-argument shift is logical.**  
   With C equal to `0x80000000`, the later four-argument dependency receives `0x00020000` in its fourth explicit position. It does not receive a sign-extended value.

3. **Selector destination can change between writes.**  
   If the first read at S + 1 returns `0x20`, the first selected byte destination is `0x31`. If dependency activity changes the selector to `0xFF` before the next read, the next destination is `0x110`.

4. **Byte truncation affects output.**  
   If T equals `0x12345678`, the first selected byte output is `0xE0`; the later selected output derived from T is `0x23`.

5. **A selector write may redirect future selector writes.**  
   If S + 1 equals `0x11` and its current byte is zero, the selected destination is the selector byte itself. A nonzero output changes the value used by the next selector read.

6. **The first loop shift argument is zero, but later ones need not be.**  
   Any initial V yields Q no greater than 31. If a later Z equals 1, the next iteration uses N equal to 4 and a second explicit argument equal to `0x00040000`.

7. **Normal dependency completion does not produce a return.**  
   If every access and dependency invocation completes normally, the represented function continues executing tail iterations. No successful completion value is specified.

## Uncertainties

The following remain **documented_only**:

- Whether the supplied representation corresponds to a valid executable path.
- The effect of invalid instruction data and omitted regions on the recovered function boundary.
- The actual calling convention and the provenance of H and L.
- The validity of the auxiliary result E across the relevant call boundary.
- Exact dependency identities and prototypes.
- Whether apparent no-argument calls have additional ABI-level inputs.
- Whether 32-bit modular arithmetic accurately describes every recovered signed operation.
- Endianness and the memory-map meaning of computed addresses.
- Dependency effects on globals, selector bytes, memory ownership, and control flow.
- Whether faults or nonlocal transfers prevent the represented infinite tail from being reached or sustained.

No application-level intent is established strongly enough to impose an additional behavioral requirement.

## Evidence

This contract is based exclusively on the supplied static representation.

The represented evidence supports the explicit widths, call ordering, store ordering, selector rereads, bit extraction, address formulas, unused explicit second argument, and unconditional tail described above. Numeric ranges and acceptance values are arithmetic deductions under the stated 32-bit interpretation.

The supplied warnings materially limit confidence in executable fidelity. No implementation inspection beyond that representation, dependency resolution, runtime execution, or runtime tests has been performed.
