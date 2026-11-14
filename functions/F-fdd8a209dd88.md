<!-- Trusted provenance: F-fdd8a209dd88; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-fdd8a209dd88

## Purpose

F-fdd8a209dd88 executes a fixed sequence of indirect calls interleaved with byte, halfword, and word memory accesses. It propagates selected call results into indexed memory, copies portions of retained words into byte-selected destinations, updates a shared control location twice, and finally attempts a call through address zero.

No higher-level purpose is established. The indexed addresses are not proven to identify objects, the byte lookups are not proven to be tables, and the shared control location is not proven to implement an operating-system operation.

This contract describes the displayed operations and their ordering. Reaching any later operation is conditional on all preceding calls and memory accesses allowing execution to continue.

## Interface

### Explicit inputs and output

The displayed interface has three 32-bit inputs and no explicit return value.

| Input | Established use |
|---|---|
| First input | No explicit use in the body. |
| Second input | Passed unchanged to F-C001. |
| Third input | Logically shifted right by 23; the resulting value, from 0 through 511, is stored as a 32-bit word. |

The first input’s lack of explicit use does not establish that unknown callees cannot observe its incoming register or stack representation.

There is no displayed success indicator, returned object, or error code. A return follows the final call, but is reached only if that call returns.

### Additional state inputs

Execution also depends on:

- An incoming link/control value used to derive F-C001’s target.
- Two incoming 32-bit machine-register values, called **R5** and **R6** in this contract.
- Unresolved secondary-register outputs of calls.
- A shared pointer-like value, **G-anchor**.
- A shared 32-bit control location, **G-control**.
- A three-byte shared value beginning at address **0x00000009**.
- Memory at dynamically computed addresses.

R5 and R6 are machine-state inputs, not additional established source-level parameters. A three-argument wrapper alone does not capture the full execution interface.

### Widths and notation

- **B(address)** means an unsigned 8-bit memory read.
- **W(address)** means a 32-bit memory read.
- **low8(value)** means the value’s least significant eight bits.
- **scale4(value)** and **scale16(value)** mean multiplication by four and sixteen.
- Unsigned right shifts are logical.
- A two-word result has a 32-bit **low** part and a 32-bit **high** part.

The representation indicates a 32-bit address and arithmetic model. Some address expressions involve signed multiplication, signed left shifts, or signed addition. Their behavior outside representable ranges is not established by a language-level reading. Claims involving such expressions are conditional on representability or on an execution model supplying the corresponding 32-bit bit-vector result.

Byte order is not established. A word read followed by low8 extraction is not interchangeable with a byte read at the same address. Likewise, the three-byte value at address 0x00000009 has a numerical value, but its byte-order mapping is unresolved.

## Behavior

### Call identification and continuation

**F-C001 through F-C126** identify call occurrences in execution order. These opaque IDs do not establish distinct target functions; different occurrences may reach the same target.

“No displayed arguments” means only that the supplied representation shows none. It does not restrict what an unknown callee can inspect through the actual calling convention.

There are no displayed loops or conditional branches between the phases below. Unknown callees can nevertheless mutate memory, fail, block, transfer control elsewhere, or not return. No unlisted callee behavior is assumed.

### 1. Initial dispatch and indexed word

F-C001 receives the second explicit input. Its target is formed from the incoming link/control value plus **0xF38**, subject to the execution model’s address arithmetic and target interpretation.

F-C002 returns a 32-bit value passed to F-C003. F-C004 through F-C006 then execute with no displayed arguments. F-C007 produces **A**.

The function next:

1. Stores the third input logically shifted right by 23 as a 32-bit word at **scale4(A) + 20**.
2. Reads **W((R5 shifted right by 22) + 20)** and retains it as **initial-word**.

The store precedes the read. If the addresses alias, the store can affect initial-word.

F-C008 receives scale4(A). F-C009 through F-C012 execute with no displayed arguments.

### 2. First byte transfer and zero argument

Define **R6-slice** as R6 logically shifted right by 22, and **initial-slice** as initial-word logically shifted right by 22.

The function reads B(R6 + 1), adds 17 to that unsigned byte, and writes low8(initial-slice) at the resulting address.

F-C013 receives three copies of an unresolved secondary-register value, followed by R6-slice. F-C014 executes with no displayed arguments. F-C015 receives R6-slice logically shifted right by 23.

For a 32-bit unsigned R6, R6-slice has at most ten significant bits. Consequently, F-C015’s displayed argument is zero.

F-C016 through F-C018 execute without displayed arguments. F-C019 produces **B-result**.

The function reads B(scale4(initial-slice) + 1) and writes a zero byte at the selected byte’s value plus 17.

F-C020 receives scale4(B-result). F-C021 through F-C023 execute without displayed arguments. F-C024 produces **C-result**, and F-C025 receives scale4(C-result) twice.

F-C026 produces **D-result**. F-C027 receives D-result, and F-C028 executes without displayed arguments.

### 3. Sign-selected clearing

The next selector-read address is:

- **1** when R6’s sign bit is clear.
- **2** when R6’s sign bit is set.

This interpretation uses the displayed signed right shift as an arithmetic sign-propagating shift.

The function reads an unsigned byte at the selected address and writes a zero byte at that value plus 17.

An unresolved secondary-register output is retained as **saved-secondary**. F-C029 receives three copies of it.

F-C030 produces a 32-bit value. F-C031 receives that value and D-result logically shifted right by 23. F-C032 through F-C034 execute without displayed arguments. F-C035 produces **E-result**.

The sign-selected byte is then read again. Its unsigned value is retained as **small-base**, and a zero byte is written at small-base + 17.

The second read is distinct from the first. Intervening calls can change the selected byte.

### 4. First control update and word-to-byte copies

F-C036 receives scale4(E-result). F-C037 through F-C039 execute without displayed arguments. F-C040 produces **H-result**, and F-C041 receives scale4(H-result) twice.

F-C042 produces **control-value**. The function writes this 32-bit value to G-control before calling F-C043 with control-value. F-C044 then executes without displayed arguments.

The function reads W(small-base + 116) and writes its low eight bits at small-base + 1.

F-C045 receives saved-secondary shifted left by 11. F-C046 produces a two-word result. F-C047 receives, in order:

1. The low word.
2. The high word.
3. control-value logically shifted right by 23.

F-C048 through F-C050 execute without displayed arguments. F-C051 produces **K-result**. Define **K-base** as scale4(K-result).

The function reads W(K-base + 116) and writes its low eight bits at K-base + 1.

### 5. Repeated read and retained payload

F-C052 receives K-base. F-C053 through F-C055 execute without displayed arguments. F-C056 produces **L-result**. Define **L-base** as L-result shifted left by two, retained as a 32-bit value.

The function again reads W(K-base + 116) and writes its low eight bits at K-base + 1. It does not reuse the previous word read.

F-C057 receives:

1. saved-secondary shifted left by 12.
2. L-base.
3. L-base again.
4. L-result shifted left by 18.

F-C058 returns a value passed to F-C059. F-C060 returns a value passed to F-C061. F-C062 through F-C064 execute without displayed arguments. F-C065 produces **M-result**.

The function reads W(scale4(M-result) + 116), retains the complete 32-bit value as **payload**, and writes low8(payload) at scale4(M-result) + 1.

F-C066 receives scale4(M-result). F-C067 through F-C069 execute without displayed arguments. F-C070 produces **N-result**.

The function stores the complete payload as a 32-bit word at scale16(N-result) + 20.

### 6. Payload replication and indexed link

F-C071 receives the result of logically shifting L-base right by two and then shifting left by 16. Bits discarded when L-base was formed must not be restored by simplifying this expression to an unchecked shift of L-result.

F-C072 returns a value passed to F-C073. F-C074 returns a value passed to F-C075. F-C076 through F-C078 execute without displayed arguments. F-C079 produces **O-result**.

The function stores payload at scale4(O-result) + 20.

F-C080 receives scale4(O-result). F-C081 through F-C083 execute without displayed arguments. F-C084 produces **Q-result**.

The function stores scale4(Q-result), as a 32-bit value, at **(L-base with its lowest two bits cleared) + 20**. Under the 32-bit shift interpretation, L-base already has those bits clear.

F-C085 receives scale4(Q-result) twice.

### 7. Zero word and retained anchor

F-C086 returns a value passed to F-C087. F-C088 returns a value passed to F-C089. F-C090 through F-C092 execute without displayed arguments. F-C093 produces **S-result**.

The function writes a zero 32-bit word at scale4(S-result) + 20.

F-C094 receives scale4(S-result). F-C095 through F-C097 execute without displayed arguments. F-C098 produces **pair-low** and **pair-high**, each 32 bits.

The function:

1. Forms **linked-value** by shifting pair-low left by two.
2. Stores linked-value at scale4(pair-high) + 20.
3. Reads G-anchor and retains its value as **saved-anchor**.

F-C099 receives linked-value, pair-high, linked-value, and linked-value, in that order.

F-C100 through F-C102 execute without displayed arguments. F-C103 produces **T-result**.

The function stores linked-value at scale4(T-result) + 20, then reads W(saved-anchor + 20) and retains it as **anchor-word**.

G-anchor is not reread for this access. Later changes to G-anchor do not redirect the read, but changes to memory at saved-anchor + 20 can affect its value. The preceding linked-value store can also affect the read if the addresses alias.

### 8. Byte transfers from retained words

F-C104 receives scale4(T-result). F-C105 through F-C107 execute without displayed arguments. F-C108 produces **tail-low** and **tail-high**, each 32 bits.

The function reads B((payload logically shifted right by 23) + 1), then writes low8(anchor-word) at the selected byte’s value plus 17.

F-C109 receives tail-high three times, followed by tail-low shifted left by two.

F-C110 returns a value passed to F-C111. F-C112 through F-C114 execute without displayed arguments. F-C115 produces **X-result**.

The function then:

1. Clears tail-low’s top two bits.
2. Logically shifts the result right by 21.
3. Adds one and reads an unsigned byte at that address.
4. Writes low8(anchor-word logically shifted right by 22) at the selected byte’s value plus 17.

Both selector-read addresses in this phase range from 1 through 512 inclusive.

F-C116 receives X-result shifted left by two.

### 9. Terminal stores, three-byte read, and final call

F-C117 through F-C119 execute without displayed arguments. F-C120 produces **Y-result**. F-C121 receives Y-result shifted left by two twice.

F-C122 executes without displayed arguments. F-C123 returns a value passed to F-C124. F-C125 produces an 8-bit result, **final-byte**, and exposes an unresolved secondary-register pointer, **terminal-pointer**.

The remaining operations occur in this order:

1. Write a zero byte at terminal-pointer.
2. Read the **three-byte shared value beginning at address 0x00000009**, covering addresses **0x00000009–0x0000000B**. Call its numerical 24-bit value **upper24**.
3. Write a 32-bit value to G-control with upper24 in bits 31–8 and final-byte in bits 7–0. Numerically, this is **upper24 × 256 + final-byte**.
4. Write a zero 16-bit halfword at **(tail-low with its top two bits cleared) logically shifted right by 7**.
5. Invoke F-C126 through address zero with three explicit zero arguments.
6. Return only if F-C126 returns.

The three-byte read occurs after the terminal-pointer byte store, not before it. If terminal-pointer identifies address 9, 10, or 11, that store can affect upper24. No decomposition into three independent byte reads, or any particular instruction-level access sequence, is established.

The halfword destination ranges from zero through 8,388,607 inclusive and need not be even-aligned.

## Side effects

### Memory accesses and aliasing

| Address pattern | Displayed access |
|---|---|
| Four-times-result + 20 | 32-bit stores |
| Sixteen-times-result + 20 | 32-bit payload store |
| Base + 116 | 32-bit reads |
| Base + 1 | 8-bit stores |
| Unsigned byte value + 17 | 8-bit stores |
| Sign-selected address 1 or 2 | 8-bit reads |
| 0x00000009–0x0000000B | Three-byte shared-value read |
| Terminal secondary-register pointer | 8-bit zero store |
| Masked tail-low shifted right by seven | 16-bit zero store |
| G-control | Two 32-bit writes |
| G-anchor | One retained pointer-like read |

A byte-selected destination ranges from address 17 through 272. These are actual displayed addresses, not implicitly relocatable structure offsets.

Access widths do not establish object sizes, allocation boundaries, or alignment guarantees. Reads and writes must retain their stated order relative to calls and other accesses. Repeated reads cannot be coalesced on the assumption that callees preserve memory.

The supplied overlapping-global warning prevents assuming that named shared locations occupy disjoint storage. In particular, the three-byte shared source, G-control, and other named storage must not be treated as independent merely because they have different names. The final G-control write consumes the three-byte value read immediately beforehand, including any effect of the preceding terminal-pointer store.

### Ownership, failure, and preconditions

No displayed operation establishes allocation, release, reference counting, ownership transfer, or restoration. Unknown callees may have such effects, but none may be assumed.

There are no displayed bounds checks, pointer checks, error branches, or rollback operations. Execution through a phase requires its reached accesses to be permitted and its preceding calls to continue. Normal return additionally requires the final zero-address call to return.

A fault does not imply rollback of earlier writes. Exact persistence and visibility depend on platform memory and fault semantics.

## Dependencies

F-C001 through F-C125 are unresolved indirect dependencies. Their displayed arguments and consumed results are specified above. Their actual identities, complete calling conventions, memory effects, ownership effects, exceptions, blocking behavior, and register preservation are **documented_only**.

F-C126 has an explicitly zero target. Whether this executes code, faults, or behaves otherwise is platform-dependent.

The apparent two-word results establish how their low and high portions are consumed, not whether the ABI regards them as single 64-bit values or paired register outputs.

## Acceptance examples

These are conditional static examples, not executed tests.

1. **Input extraction.** Third input 0xFFFFFFFF produces first-store value 511; third input 0x007FFFFF produces zero. The destination still depends on F-C007.

2. **Initial aliasing.** If the first store and initial-word read use the same address under ordinary coherent memory semantics, the read observes the stored value.

3. **Zero argument.** For every unsigned 32-bit R6, F-C015 receives zero as its displayed argument.

4. **Sign selection.** R6 values 0x7FFFFFFF and 0x80000000 select addresses 1 and 2 respectively. A selected byte of 255 causes a zero-byte destination of 272.

5. **Repeated lookup.** If an intervening callee changes the sign-selected byte, the two clears can target different addresses. The second lookup must not reuse the first value.

6. **Repeated word read.** If W(K-base + 116) changes between its two reads, the byte writes at K-base + 1 can differ.

7. **Payload width.** Payload 0x123456AB yields an associated byte write of 0xAB, while both later payload stores retain 0x123456AB.

8. **Saved anchor.** Changing G-anchor after capture does not redirect the anchor-word read. Changing memory at saved-anchor + 20 before that read can affect it.

9. **Final numerical concatenation.** If the three-byte read yields upper24 = 0x123456 and final-byte = 0xAB, the final G-control value is 0x123456AB, irrespective of the unresolved memory byte order.

10. **Terminal source overlap.** If terminal-pointer is address 10, the zero-byte store precedes the three-byte read covering addresses 9–11. Under ordinary coherent memory semantics, that read includes the cleared byte. Which numerical bits this changes depends on byte order.

11. **Halfword extremes.** Tail-low zero selects address zero. Tail-low 0xFFFFFFFF selects 0x007FFFFF, an odd address. Success depends on mapping and alignment rules.

12. **Final-call fault.** If the zero-address call faults, the preceding terminal store, three-byte read, G-control write, and halfword store have already been reached in that order. No return or rollback follows from the contract.

## Uncertainties

**Established static evidence:** explicit input widths, displayed operation order, call argument duplication, retained values, access widths, masks, shifts, two G-control writes, the three-byte source address and its terminal ordering, and the final zero-address call.

**Inferred intent:** repeated indexed stores and word-to-byte transfers suggest structured state manipulation. No object model, protocol, or subsystem purpose follows.

**documented_only:**

- Exact ABI, target-address interpretation, and register preservation.
- Overflow behavior and machine-level interpretation of signed arithmetic and shifts.
- Byte order, alignment rules, permissions, and memory mapping.
- Volatile or memory-mapped-device behavior.
- Exact overlap among named shared locations.
- Physical access decomposition or atomicity of the three-byte read.
- Unknown callee effects and whether normal completion is possible.

## Evidence

This contract derives from the supplied static source representation and its overlapping-global warning. The prior draft and reviewer feedback are not independent evidence.

The terminal expression establishes a three-byte value sourced at address 0x00000009, read after the secondary-register-pointer zero-byte store and numerically concatenated above the 8-bit final result before the G-control write. It does not establish three separate byte-load instructions.

No runtime execution or tests have been performed. All acceptance examples are conditional consequences of the displayed operations, not observed results.
