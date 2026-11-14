<!-- Trusted provenance: F-3fb997e0e3dc; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-3fb997e0e3dc

## Purpose

F-3fb997e0e3dc describes a represented sequence of indirect calls, calculated-address memory accesses, byte writes, and 32-bit writes. The supplied representation ends at an invalid-instruction boundary. It does not establish successful completion or a normal return.

No application-level purpose is established. Repeated scaling and recurring offsets suggest structured-data manipulation, but that interpretation is **inferred intent**, not evidence of particular objects, fields, or runtime services.

The contract specifies static dataflow and represented event order. It does not establish that the apparent entry point, call boundaries, or terminal boundary correspond to valid, reachable executable instructions.

## Interface

### Inputs and required state

Two explicit inputs are represented, each 32 bits wide. Both are forwarded unchanged as the first two arguments of F-A1. Their signedness, meaning, and ownership are unresolved.

Additional entry-state dependencies are:

- **Entry R7:** a 32-bit unsigned value. Its logical right shift by two supplies F-A1’s third argument. Its original value supplies a later word write.
- **Entry R6:** an address-valued register supplying the base for an early byte write at offset `0x01`.
- **Entry link register:** supplies the base of F-A1’s calculated target.
- **Memory and executable mappings:** must support the represented accesses and indirect transfers for traversal to continue.

A later call also consumes a secondary register-derived 32-bit unsigned value whose defining producer is unresolved. It is not established as an explicit input, an entry-state value, or any particular dependency’s primary result.

These dependencies do not establish a complete calling convention.

### Outputs and termination

No normal return value or successful-return condition is established.

Several dependency results feed later operations. Three results are represented as 64-bit quantities and consumed through their low and high 32-bit halves. This representation does not prove whether the executable convention returns a single 64-bit value or multiple register values.

The represented endpoint is an invalid-instruction boundary. Actual exception delivery, fault handling, recovery, and subsequent execution remain unresolved.

### Widths and arithmetic

A **byte** contains eight bits; a **word** contains 32 bits. Byte writes retain the low eight bits of their supplied value. Word writes retain 32 bits. Explicit byte reads are unsigned and therefore produce values from zero through 255.

All formulas below describe 32-bit bit-vector calculations: additions, products, and left shifts retain their low 32 bits. Right shifts of unsigned quantities are logical. For each represented 64-bit result, **low** and **high** mean its low and high 32-bit halves.

This is a width-explicit account of represented calculations, not a claim about portable signed arithmetic or verified processor behavior. Endianness, alignment requirements, instruction mode, and actual fault semantics are unresolved.

Continued traversal is conditional on dependencies returning and on the required mappings and accesses being valid. No explicit validation of these conditions appears.

## Behavior

### Event-order convention

The numbered stages define the order of represented calls and explicit memory accesses. Prose order within each stage is significant. This order does not establish runtime volatility, atomicity, memory ordering, or external observability.

Dependency IDs identify **call occurrences**, not necessarily different callees. A range such as F-A2–F-A5 denotes four successive calls. “No explicit arguments” means the supplied representation lists none; it does not prove the absence of relevant register or other machine state.

Capital-letter labels below are contract notation for retained values, not implementation identifiers or object types. Retained dataflow does not imply that dependencies preserve globals or memory.

### 1. Entry forwarding and first word write

F-A1 receives the two explicit inputs followed by entry R7 logically shifted right by two. Its calculated target is **entry link register + `0x0ffe`**, reduced to 32 bits.

F-A2–F-A5 follow without explicit arguments. Let **A** be F-A5’s word result. The next explicit access writes the original entry R7 word to **4A + `0x14`**.

### 2. Unmasked address calculation followed by masking

F-B1 receives **4A**. F-B2–F-B5 follow without explicit arguments; let **B** be F-B5’s word result.

Define **S = 4B**. A word write stores S at **16B + `0x14`**. Only afterward is **M = B & `0x3fffffff`** formed, followed by **Q = M >> 2**.

The write therefore uses the unmasked B in its address calculation. The subsequent mask does not alter that earlier event.

### 3. Second indexed word write

F-C1 receives M. F-C2–F-C5 follow without explicit arguments; let **C** be F-C5’s word result.

A word write stores M at **4C + `0x14`**.

### 4. Entry-base byte write and calculated word destination

F-D1 receives **4C**. F-D2–F-D5 follow without explicit arguments; let **D** be F-D5’s word result.

Define **E = 8C**. A byte write stores the low eight bits of E at **entry R6 + `0x01`**. The next explicit access is a word write of **D << 2** at **E + `0x34`**.

### 5. First split result and later write to the same word address

F-E1 receives **D << 2**. F-E2 follows without explicit arguments; let its word result be **T**. Define **N = D & `0x3fffffff`**.

F-F1 receives T. F-F2–F-F5 follow without explicit arguments; let F-F5’s represented 64-bit result be **P**.

A byte write stores the low eight bits of E at **N + `0x01`**. A word write then stores **high(P)** at **E + `0x34`**, the word destination used in stage 4.

Both word writes remain distinct in the represented sequence. Intervening dependencies have unresolved effects, so the contract does not assume that the earlier value remains present until the later write.

### 6. Byte writes around two argument-bearing calls

F-G1 receives **low(P) << 2**. F-G2–F-G5 follow without explicit arguments; let their final word result be **G**.

Define **H = 16C**. A byte write stores the low eight bits of H at **(N >> 20) + `0x11`**. Another byte write stores the low eight bits of S at **H + `0x0d`**.

F-H1 receives two equal arguments, both **G << 2**. F-H2 follows without explicit arguments; let its word result be **J**.

F-I1 receives J and **J >> 3**, in that order. F-I2–F-I5 follow without explicit arguments; let the final word result be **K**.

A byte write stores the low eight bits of H at absolute address **`0x11`**. Another byte write stores the low eight bits of S at **H + `0x0d`**. This second write to the same calculated destination remains a separate represented operation even though its supplied value matches the earlier one.

### 7. Further base calculations and writes

F-J1 receives **K << 2**. F-J2–F-J5 follow without explicit arguments; let the final word result be **L**.

A word write stores S at **N + `0x14`**. A byte write then stores the low eight bits of **N << 1** at **N + `0x0d`**.

Define:

- **X = 32L**;
- **Y = L << 7**;
- **Z = (L & `0x07ffffff`) >> 17**.

F-K1–F-K3 execute without explicit arguments; let F-K3’s word result be **V**.

A word write stores Y at **X + `0x14`**. A byte write then stores the low eight bits of **V << 2** at **X + `0x0d`**.

Z ranges from zero through 1023; consequently, **Z + 7** ranges from `0x07` through `0x406`.

### 8. Two fresh byte reads separated by calls

F-L1 receives **V << 2**. F-L2–F-L5 follow without explicit arguments.

A byte write stores the low eight bits of **Q << 1** at **Z + 7**. An unsigned byte is then read from **X + `0x01`**. Its numeric value plus `0x11` supplies the destination for a byte write of the low eight bits of Z.

F-M1 receives three equal arguments, each **V << 21**. F-M2 follows without explicit arguments; let its word result be **W**.

F-N1 receives W. F-N2–F-N5 follow without explicit arguments; let the final word result be **AA**.

A second unsigned-byte read occurs at **X + `0x01`**. Retain this second value as **b**. A byte write stores zero at **b + `0x11`**.

The two reads are separate accesses. The second value must not be substituted with the first: intervening dependencies have unresolved memory effects, and aliasing with other accesses is not excluded.

### 9. Saved-byte effects and unresolved secondary register value

F-O1 receives **AA << 2**. F-O2–F-O5 follow without explicit arguments.

Define **U** as the low eight bits of **W >> 3**, and **R = 2b**. A byte write stores U at **X + `0x0d`**. A byte write then stores b at absolute address **`0x0d`**.

F-P1 receives R, `0x98`, and `0x260`, in that order. F-P2 follows without explicit arguments.

F-Q1 receives the unresolved secondary register-derived unsigned value logically shifted right by three. Its producer must not be inferred from the listed primary results. F-Q2–F-Q5 follow without explicit arguments; let the final word result be **AB**.

F-R1 receives **AB << 2**; let its word result be **AC**.

A byte write stores U at **R + `0x0d`**. Another byte write stores `0xc0` at absolute address **`0x0d`**.

R ranges from zero through 510. These later operations reuse retained U and b; they do not reread the memory from which b was obtained.

### 10. Loaded-word chain and second split result

F-S1 receives **AC << 2**. F-S2–F-S5 follow without explicit arguments; let the final word result be **AD**.

A word write stores **AD << 2** at **R + `0x24`**. A word read from **R + `0x74`** supplies retained value **F**. Define **I = AD << 5**.

F-T1 receives F, **F << 1**, I, and **(F & `0x7fffffff`) >> 2**, in that order. F-T2–F-T5 follow without explicit arguments.

A word write stores I at **F + `0x24`**. A fresh word read from **F + `0x74`** supplies F-U1’s argument. This read is distinct from the earlier read at **R + `0x74`**, even if the addresses happen to coincide.

F-U2–F-U5 follow without explicit arguments; let the final represented 64-bit result be **P2**. Define **J2 = low(P2) << 2**.

A word write stores the complete 32-bit value **W >> 3** at **4 × high(P2) + `0x14`**. Unlike U, this stored value is not truncated to eight bits.

### 11. Third split result and indexed word write

F-V1 receives J2, J2, J2, and **high(P2) >> 3**, in that order. F-V2 follows without explicit arguments; let its represented 64-bit result be **P3**.

F-W1 receives **low(P3)**, **high(P3)**, **low(P3)**, and **high(P3) >> 3**, in that order. F-W2–F-W5 follow without explicit arguments; let the final word result be **AE**.

A word write stores J2 at **4AE + `0x14`**.

### 12. Repeated byte-indexed writes

F-X1 receives **4AE**. F-X2–F-X5 follow without explicit arguments; let the final word result be **AF**. Define **AG = AF << 2**.

An unsigned byte is read from **F + `0x01`**. A byte write stores the low eight bits of AG at that read byte’s numeric value plus `0x11`.

F-Y1–F-Y3 execute without explicit arguments; let the final word result be **AH**.

Another unsigned byte is read from **F + `0x01`**. A byte write stores the low eight bits of **(AF & `0x3fffffff`) >> 20** at this newly read byte’s numeric value plus `0x11`.

F-Z1 receives **AH << 2**. F-Z2–F-Z5 follow without explicit arguments; let the final word result be **AI**.

An unsigned byte is read from **1024AG + `0x01`**. A byte write stores zero at that read byte’s numeric value plus `0x11`.

Every described read is a fresh represented memory access.

### 13. Final calls and represented boundary

F-AA1 receives **AG << 3**, **AG << 3**, AG, and **AI << 2**, in that order. F-AA2–F-AA5 follow without explicit arguments; let the final word result be **AJ**.

A fresh unsigned-byte read occurs at **1024AG + `0x01`**. A byte write stores zero at that read byte’s numeric value plus `0x11`. The prior zero write does not eliminate this read or write from the represented sequence.

F-AB1 receives **AJ << 2**. F-AB2–F-AB5 follow without explicit arguments.

The representation then reaches its invalid-instruction boundary. No normal completion, recovery branch, or post-boundary state is established.

## Side effects

Established static effects are the ordered dependency calls and explicit accesses described above. Their external runtime observability is not established.

Recurring offsets describe access locations only. They do not establish field layouts, object sizes, ownership, or disjoint storage.

Important address and sequencing consequences are:

- Unsigned-byte-indexed destinations of the form **read byte + `0x11`** range from absolute address `0x11` through `0x110`.
- Absolute addresses `0x0d` and `0x11` receive explicit byte writes.
- **(N >> 20) + `0x11`** is a different calculation, ranging from `0x11` through `0x410`.
- Calculated addresses may alias other accesses, dependency-visible storage, low addresses, or executable mappings.
- No explicit bounds, null, alignment, or error-result checks appear.
- No explicit transaction or rollback appears in the supplied representation. Fault handling, dependency behavior, and executable control flow remain unresolved; preservation of earlier effects after a later fault is not guaranteed.
- Separate ordered writes, including equal-value writes, must remain distinct in the represented event sequence. This does not assert verified external observability, volatility, atomicity, or memory-ordering guarantees.

No explicit allocation, release, ownership transfer, reference-count change, or storage-lifetime guarantee is established. Unknown dependencies may affect memory, globals, mappings, resources, or control flow; no preservation or ownership behavior is assumed.

## Dependencies

Opaque dependency IDs distinguish occurrences while retaining established target relationships.

| Occurrence | Established calculated target |
|---|---|
| F-A1 | Entry link register + `0x0ffe`, reduced to 32 bits |
| F-F5 | `0x0882ef05 + 0x0ffe = 0x0882ff03` |
| F-R1 | `0x0882f0fb + 0x0e08 = 0x0882ff03` |

F-F5 and F-R1 calculate the same target address. This equality is established arithmetic, not an unresolved callee-identity question.

Equal addresses do not prove equal executable contents, instruction mode, mappings, or behavior at the two call times. Intervening dependencies have unresolved effects.

Other occurrences also use represented constant-derived or address-derived indirect targets. Distinct occurrence IDs do not imply distinct addresses.

No dependency is established as an allocator, collector, dispatcher, encoder, or other application primitive. Complete argument conventions, return conventions, side effects, ownership behavior, and termination guarantees remain unresolved.

## Acceptance examples

These are **conditional static examples**, not runtime tests. Each assumes traversal reaches the stated events with returning dependencies and valid required accesses. They describe represented calculations and order, not guaranteed externally observed outcomes.

1. **Entry-dependent target and argument.** With entry link register `0x10000000`, F-A1’s calculated target is `0x10000ffe`. With entry R7 equal to `0x0000000b`, its third argument is `2`. Executability of the target is not implied.

2. **Retained entry word.** If entry R7 is `0x0000000b` and F-A5 returns `3`, the next explicit word write supplies `0x0000000b` to address `0x20`.

3. **Equal calculated targets.** F-F5 and F-R1 both calculate `0x0882ff03`. Different dependency behavior at those occurrences would not alone contradict the contract because surrounding state, code contents, and mappings are unresolved.

4. **Masking follows the write.** If B is `0x40000001`, S is `4`, and **16B + `0x14`** reduces to `0x24`. The word write of `4` at that address precedes formation of M = `1` and Q = `0`.

5. **Byte truncation differs from word storage.** If W is `0x00000800`, U is zero. The later word write derived from **W >> 3** instead supplies `0x00000100`.

6. **Fresh reads may differ.** If the first read at **X + `0x01`** yields `5`, its indexed write targets `0x16`. If the second yields `9`, the zero write targets `0x1a`, b is `9`, and R is `18`. F-P1 subsequently receives `18`, `0x98`, and `0x260`.

7. **Aliasing does not merge represented operations.** If N equals entry R6, both byte writes at offset `0x01` remain in their specified positions. Their existence does not establish that an external observer can distinguish them.

8. **No fault-survival guarantee.** If traversal encounters a fault after earlier represented writes, the representation supplies no explicit rollback operation. Whether those writes persist or become externally visible remains unresolved.

9. **No successful-return implication.** Even with returning dependencies and valid listed accesses, the represented endpoint is an invalid-instruction boundary, not an established normal return.

## Uncertainties

The following remain **documented_only**:

- Whether the sequence is valid executable code and begins at a genuine function entry.
- Whether the apparent calls and terminal boundary are correctly represented and reachable.
- Calling convention, register-result convention, instruction mode, byte order, alignment, and actual fault behavior.
- Executable mappings and callee identities, including at the shared calculated target `0x0882ff03`.
- The producer of the secondary register-derived value consumed by F-Q1.
- Validity and application meaning of low-address accesses.
- Processor arithmetic semantics beyond the explicit bit-vector account.
- Dependency effects, preservation guarantees, ownership behavior, and termination.
- Fault recovery, persistence of prior effects, external observability, concurrency, interrupts, atomicity, and memory ordering.
- Application-level layouts, success criteria, and error protocols.

Possible structured-runtime intent remains inference. It does not justify inventing object types, allocation behavior, validation, rollback, or recovery.

## Evidence

Established static evidence consists of the supplied representation’s explicit input widths, register dependencies, retained dataflow, ordered calls and accesses, access widths, address expressions, constants, masks, shifts, split-result uses, and terminal invalid-instruction warning.

The first target’s link-register addition and the equal calculated targets of F-F5 and F-R1 follow from explicit arithmetic. Unknown calling conventions and mappings do not erase those relationships.

The invalid-instruction warnings limit conclusions about executable control flow and recovered call boundaries. Separate represented operations establish sequence structure, not verified runtime observability or fault-survival guarantees.

No runtime tests have been performed. No implementation access, dependency inspection, independent instruction analysis, or dynamic validation was available.
