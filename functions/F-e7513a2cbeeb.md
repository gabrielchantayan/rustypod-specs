<!-- Trusted provenance: F-e7513a2cbeeb; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-e7513a2cbeeb

## Purpose

F-e7513a2cbeeb performs an ordered sequence of memory reads and writes, derives addresses from argument bits and memory contents, and finally transfers control through a memory-loaded target.

The established behavior is memory manipulation followed by an indirect invocation. No higher-level purpose, object model, allocation protocol, or meaningful success condition is established. In particular, the byte-sized values used as addresses must not be assumed to be object handles or valid pointers.

This contract distinguishes:

- **Established static evidence:** operations and ordering represented in the supplied source.
- **Inferred machine interpretation:** 32-bit wrapping arithmetic and address calculations consistent with the represented operations.
- **Documented-only unknowns:** matters not resolved by the supplied evidence, including calling convention, memory validity, and the indirect target’s behavior.

No runtime tests have been performed.

## Interface

### Explicit inputs

The function has four explicit argument positions:

| Position | Represented width and interpretation | Established use |
|---|---|---|
| First | 32-bit value with unspecified signedness | No explicit use in the supplied body |
| Second | Signed 32-bit integer | Arithmetic right shift determines the address of the first write |
| Third | Unsigned 32-bit integer | Logical right shift contributes to a computed offset |
| Fourth | Signed 32-bit integer | Arithmetic right shift contributes to the same computed offset |

The unused first argument is not established to be ignored by every possible execution environment. The narrower claim is that the supplied body contains no explicit operation using it.

### Implicit inputs

The function also depends on:

- An incoming register-derived base, denoted **R**, that is not represented as an explicit argument.
- Three fixed global locations containing 32-bit values, denoted **G-selector**, **G-base**, and **G-payload**.
- Byte-addressed memory at absolute addresses `0xC4` and `0xCB`.
- A further fixed global 32-bit value, denoted **G-call**, read for the final invocation.
- All memory reached through subsequently loaded addresses and offsets.

These global labels describe distinct source references, not guaranteed-disjoint storage. Earlier writes can affect later reads if their addresses overlap.

### Output and completion

There is no represented return value.

The body ends with an indirect invocation and a subsequent return. The supplied recovery does not establish whether the machine-level transfer is an ordinary call, a tail transfer, or a more complex control-flow construct. If the target returns normally to the represented continuation, the function returns without producing an explicit result.

There is no represented status result, validation failure result, or exception handler.

### Preconditions for uninterrupted execution

Normal completion requires every executed memory access to be permitted at its actual address and width, and the final target to support the actual transfer and calling convention. Alignment requirements, if any, are platform-dependent and unknown.

No input checks establish these conditions. A caller cannot assume that providing valid explicit arguments alone is sufficient; the implicit base, globals, low-address memory, and pointer-linked memory must also be suitable.

## Behavior

### Arithmetic conventions

The supplied representation uses 8-bit bytes, 16-bit halfwords, and 32-bit integers. Address calculations are described below under a 32-bit wrapping machine interpretation. Signed right shifts are arithmetic; unsigned right shifts are logical.

These conventions describe the apparent machine behavior, not portable signed-arithmetic semantics. Overflow behavior and the accuracy of the recovered types remain uncertainties.

For describing the computed offset, let:

- \(U\) be the third argument shifted logically right by 12.
- \(V\) be the fourth argument shifted arithmetically right by 11.
- \(X\), under 32-bit wrapping arithmetic, be
  \[
  X = 64\bigl(U-(2^{24}V+4)-79\bigr).
  \]

The represented calculation interprets the resulting 32-bit pattern as signed, shifts it arithmetically right by one, doubles the result, subtracts `0x72`, and rotates the resulting unsigned word right by its own low five bits.

Because \(X\) is a multiple of 64, the right-shift-and-double operation restores its bit pattern under this interpretation. Consequently, the pre-rotation word is \(X-114\), its low five bits equal 14, and the computed offset **O** is:

\[
O=\operatorname{ROR}_{32}(X-114,14).
\]

This simplification is conditional on the stated 32-bit interpretation. The source representation is not sufficient to establish portable behavior for signed overflow.

### 1. Initial write and computed-offset writes

The first observable operation is a 32-bit write of the value 13 to the address obtained by arithmetically shifting the second argument right by 21 and adding 13.

The function then performs these global reads, in order:

1. Read **G-selector** as a 32-bit word.
2. Read **G-base** as a 32-bit word.
3. Read **G-payload** as a 32-bit word.

It calculates **O** from the third and fourth arguments.

Next, it reads **G-base** again and writes that newly read 32-bit value at address **R + `0x380` + O**.

It then writes the low 16 bits of the previously captured **G-payload** value at address **O + captured G-base**.

The two reads of **G-base** have different roles: the earlier value supplies the halfword destination base; the later value supplies the word being written. They must not be treated as one interchangeable read. The initial write can also affect the global values if its destination aliases their storage.

### 2. Capture byte values and derive a small address base

The function next performs the following operations in the represented order:

1. Read one byte from absolute address `0xC4`; retain this first sample.
2. Form **K** by taking the most significant byte of the captured **G-selector** word and multiplying it by four.
3. Read one byte from absolute address `0xCB`; zero-extend it to form **Q**.
4. Read one byte from absolute address `0xC4` again; zero-extend this second sample to form **P**.
5. Read one byte from **R + `0x393`**; call this value **A**.
6. Read one byte from **Q + 6**; call this value **B**.
7. Read one byte from **K + 2**; call this value **C**.

Here **A**, **B**, **C**, **P**, and **Q** range from 0 through 255. **K** is a multiple of four between 0 and 1020.

The two reads from `0xC4` remain distinct observations. Their equality is not guaranteed by this contract, particularly for externally changing or device-backed memory.

### 3. Perform six byte writes

Using the captured values, the function performs these writes in order:

| Order | Destination | Written byte |
|---|---|---|
| 1 | **C + `0x1D`** | **A** |
| 2 | **K + `0x19`** | **B** |
| 3 | **K + `0x14`** | **C** |
| 4 | **P + `0x0E`** | First sample from address `0xC4` |
| 5 | **B + 9** | **C** |
| 6 | **A + 3** | `0xAC` |

These writes use captured values; they do not reread their source bytes.

All six destinations lie in low-address memory under the stated value ranges. Their validity is not established. Destinations may overlap one another or memory that is read later.

### 4. Load linked values and perform four word writes

After the six byte writes, the function reads:

1. A 32-bit word at **Q + `0x5C`**, denoted **L**.
2. A 32-bit payload at **A + `0x40`**, denoted **W**.
3. A 32-bit word at **C + `0x24`**, denoted **N**.

It then performs these 32-bit writes, in order:

| Order | Destination | Written word |
|---|---|---|
| 1 | **L + `0x6C`** | **K** |
| 2 | **C + `0x4C`** | Zero-extended **Q** |
| 3 | **N + `0x2C`** | The 32-bit pattern **L** |
| 4 | **C + `0x0C`** | Captured payload **W** |

The preceding byte writes may alter these word reads through overlap. Likewise, these word writes may alter subsequent byte, halfword, pointer, or global reads.

Although **L** and **N** are represented as signed integers, they participate in address calculations. No object type or ownership meaning is established for either value.

### 5. Capture final dispatch inputs and perform three writes

The function next reads:

1. One byte at **N + L**, denoted **E**.
2. One 16-bit value at **K + L**, denoted **H**.
3. A code-target pointer at **K + Q**, denoted **T**.

The pointer width is inferred to be 32 bits from the represented environment; the supplied text does not independently establish the machine ABI.

After capturing **T**, the function reads a signed 16-bit value at **L + `0xAC`**, denoted **S**, and performs:

1. A byte write at **N + S**, storing the low eight bits of **N**.
2. A halfword write at **T + N**, storing the low 16 bits of **T**.
3. A word write at **N + E**, storing **E** zero-extended to 32 bits.

The signedness of **S** matters: a halfword containing `0xFFFF` supplies an offset of −1, not 65535.

**T** is captured before these final three writes. Overwriting its original pointer slot does not itself cause a reread. However, writes into memory containing the target’s instructions or data could affect the later transfer or execution.

### 6. Invoke the captured target

The final indirect invocation supplies three represented arguments:

1. **K**
2. A 32-bit value read from **G-call** after the preceding writes
3. The captured 16-bit value **H**

No signed interpretation of **H** is established. Its extension into an argument register or stack slot is ABI-dependent.

The final target is not checked for nullness, alignment, executability, or signature compatibility.

## Side effects

The represented body contains:

- Sixteen explicit memory writes: seven byte writes, two halfword writes, and seven word writes.
- Reads from fixed globals and absolute low addresses.
- Reads and writes through an implicit register-derived base.
- Pointer-linked memory accesses whose addresses depend on previously read values.
- One indirect control transfer with unknown additional effects.

The count describes operations in an uninterrupted execution, not distinct addresses or guaranteed completed writes.

Writes are ordered and nontransactional. A fault or failed transfer can occur after earlier writes have taken effect. No rollback, cleanup, or recovery path is represented.

No allocation, deallocation, reference counting, locking, or ownership transfer is directly established. Memory remains unclassified as borrowed, owned, shared, device-backed, or executable. The indirect dependency must not be assumed to preserve globals, retain no arguments, or avoid releasing memory.

## Dependencies

### Indirect dependency: F-unknown-indirect

This document uses **F-unknown-indirect** as an opaque reference for the unresolved target loaded from **K + Q**. It is not a resolved function identity supplied by the dependency map.

Established interaction:

- Its target is read before the final three writes.
- It is invoked after those writes.
- It receives the represented arguments **K**, the late **G-call** value, and **H**.

Documented-only unknowns:

- Actual identity and possible target set.
- Calling convention and argument extension rules.
- Whether the transfer returns.
- Return-value meaning, if any.
- Global preservation, memory ownership, synchronization, and fault behavior.
- Whether code memory modified earlier is relevant to its execution.

No other direct function dependency is established.

## Acceptance examples

These examples are conditional expectations derived from static evidence, not executed tests.

### Initial address calculation

If the second argument is zero, the first operation writes the 32-bit value 13 at address `0x0000000D`.

If the second argument is −1 and arithmetic right shift applies, the destination is address `0x0000000C`.

Both expectations require that the relevant write is permitted. Neither example implies that such low addresses are valid on the deployment platform.

### Computed offset

If the third and fourth arguments are zero, then under 32-bit wrapping arithmetic:

- \(X=-5312\).
- The pre-rotation word is `0xFFFFEACE`.
- The rotation count is 14.
- **O** is `0xAB3BFFFF`.

The computed word destination is therefore **R + `0x380` + `0xAB3BFFFF`**, with 32-bit address wrapping. The halfword destination uses the earlier captured **G-base** value instead of **R**.

### Distinct repeated reads

If the first read from `0xC4` produces 7 and the second produces 9, the fourth byte write targets address 23 and stores 7.

This distinguishes the stored first sample from the destination-forming second sample.

### Truncation and signed offset

If **N** has bit pattern `0x12345678` and **S** is −2, the final byte write targets **N − 2** and stores `0x78`.

If **T** has bit pattern `0x89ABCDEF`, the subsequent halfword write stores `0xCDEF`, not the full target value.

### Late global observation

If a preceding write changes the storage of **G-call**, the final invocation receives the value visible at the late global read, not a value captured on entry.

Conversely, **H** is captured before the final three writes. A later overwrite of its source memory does not change the represented third argument.

### Partial execution

If the first six byte writes succeed but the following read at **Q + `0x5C`** faults, the earlier writes remain observable unless the platform independently provides stronger transactional behavior. The body itself supplies no rollback.

## Uncertainties

The following are **documented_only**:

- The true ABI, architecture, endianness, and alignment rules.
- The origin and validity of **R**.
- Whether all recovered argument types and operation boundaries are accurate.
- Whether signed arithmetic is faithfully represented by 32-bit wrapping operations.
- Whether any accessed locations are volatile, device-backed, concurrently modified, or mutually aliased.
- The exact pointer width and interpretation of arithmetic on the code target.
- The behavior of invalid, misaligned, protected, or unmapped accesses.
- The accuracy of the recovered stack and control flow.
- Any application-level meaning for the globals, offsets, payloads, or final invocation.

The supplied recovery includes warnings about possible stack-pointer effects, overlapping global symbols, and unresolved indirect control flow. These limit confidence in a complete ABI-level contract. They do not justify inventing missing validation, ownership rules, or a conventional dispatch-table abstraction.

## Evidence

All established claims derive from the supplied static representation of F-e7513a2cbeeb.

The evidence supports the explicit access widths, represented operation order, argument shifts, truncating stores, repeated reads, and indirect invocation described above. The rotation simplification and numerical example are mathematical deductions conditional on the stated 32-bit machine interpretation.

No implementation access, external dependency inspection, disassembly verification, execution, or runtime testing was performed. No snapshot identifier is asserted.
