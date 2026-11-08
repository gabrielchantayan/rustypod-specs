<!-- Trusted provenance: F-9ef5d48fbb2f; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-9ef5d48fbb2f

## Purpose

F-9ef5d48fbb2f performs a conditional loop containing indirect calls, arithmetic-derived memory accesses, and two exit paths that assemble a 64-bit result.

The supplied static representation establishes call order, direct access widths, arithmetic expressions, and control flow. It does not establish a domain-level purpose, an object model, or a protocol.

This contract distinguishes:

- **Established static evidence:** operations and ordering explicitly represented.
- **Inferred execution model:** 32-bit machine-word arithmetic where signed overflow, shifting, or address formation requires qualification.
- **Unresolved behavior:** dependency effects, auxiliary-register provenance, memory validity, and actual register preservation.

No runtime tests have been performed.

## Interface

### Explicit inputs

| Input | Width and interpretation | Uses |
|---|---|---|
| First input | Unsigned 32-bit value | Contributes to the first call’s first argument and supplies a doubled argument. Replaced during a continuing iteration. |
| Second input | Signed 32-bit value | Passed to the first call; used in a quadratic expression and an absolute address calculation. Replaced after the last call of a continuing iteration. |

Neither input is validated before use.

### Implicit inputs

The routine also depends on:

- A register-derived additive value used in the first call’s first argument.
- Incoming link/dispatch state used to select the first call target.
- Three auxiliary values represented as secondary register outputs.
- A global gate.
- A global 32-bit dependency argument.
- A global 32-bit XOR operand.
- A global 32-bit return word.
- Stack storage whose address is passed to a dependency.

The auxiliary values are distinguished here by role:

- **Primary auxiliary value:** used in the gate-zero return, an early body call, an address calculation, and subsequent arithmetic.
- **Late-call auxiliary value:** passed to a call near the end of a continuing iteration.
- **Iteration auxiliary value:** used to replace the second input after the last call.

These names do not establish additional formal parameters, producer identities, or stable register lifetimes. In particular, the representation does not independently establish which calls produce or preserve these values.

### Output

The represented result is 64 bits, composed of two 32-bit halves:

| Half | Value |
|---|---|
| Upper | Global return word minus `0x54` |
| Lower | Twice the exit-selected 32-bit value |

The exit-selected value is:

- The primary auxiliary value shifted right by nine positions on the gate-zero path.
- The low 32 bits of F-X05’s result on the sign-test exit.

The upper half is read from the global return word during final construction. It is not taken from F-X05’s upper result half.

No explicit error code, exception translation, or failure-status convention is shown.

### Widths and arithmetic assumptions

The representation contains 32-bit integer calculations and address expressions, 32-bit and 16-bit stores, a one-byte load, and a 64-bit result.

Unsigned operations support a modulo-\(2^{32}\) interpretation. Signed multiplication, addition, subtraction, and left shifts may exceed the signed 32-bit range. The representation alone does not establish their concrete execution semantics in every such case.

For conditional arithmetic examples, this contract uses the following **inferred word model**:

- Arithmetic retains the low 32 bits.
- Signed comparisons interpret those bits as a signed 32-bit value.
- Signed right shifts propagate the sign bit.
- Address calculations retain a 32-bit address representation.
- A 16-bit store retains the low 16 bits.

This model is not a claim that overflowing signed expressions have portable source-language behavior.

## Behavior

### 1. Iteration entry and initial calls

Each iteration uses the current first and second inputs.

Before making any call, the routine computes:

- Twice the first input.
- The second input multiplied by two, then by the second input, then by two.

Under the word model, the quadratic result is \(4s^2 \bmod 2^{32}\), where \(s\) is the current second input. Its evaluation occurs before the gate is read, even if the iteration subsequently exits through the gate-zero path.

The routine then makes these calls in order:

1. **F-X01**, with:
   - First input plus the register-derived additive value.
   - Second input.
   - Twice the first input.

   Its target is formed from the represented link/dispatch state plus `0xaaa`.

2. **F-X02**, with the value read from the global dependency-argument location after F-X01 returns.

3. **F-X03**, with no explicit arguments shown.

Only after F-X03 returns does the routine read and test the global gate.

The gate is not an entry precondition. F-X01, F-X02, and F-X03 run before each gate decision, and their effects may influence the value subsequently read.

### 2. Gate-zero exit

If the gate is zero, execution leaves the loop without the direct body stores described below.

The routine:

1. Shifts the primary auxiliary value right by nine positions.
2. Doubles that result for the lower return half.
3. Reads the global return word and subtracts `0x54` for the upper return half.
4. Returns the assembled 64-bit value.

Under the word model, the lower half is:

\[
2\,\operatorname{ASR}_{32}(\text{primary auxiliary value},9)
\bmod 2^{32}.
\]

For negative values, an arithmetic right shift is not equivalent to unsigned division by 512.

### 3. Nonzero gate: first stores and result-producing call

If the gate is nonzero, the routine stores the quadratic result as 32 bits at:

\[
2s + 0x34.
\]

This is an absolute computed address in the representation. No separately supplied object base is added.

It then calls:

1. **F-X04**, passing:
   - Twice the first input.
   - The primary auxiliary value.
   - An address of local stack storage.

2. **F-X05**, with no explicit arguments shown, receiving a represented 64-bit result.

The low 32 bits of F-X05’s result become the candidate used if the next conditional exit is taken. Its upper 32 bits are used only if execution continues past that condition.

The routine next forms an adjusted value equal to the primary auxiliary value plus 10, and stores the quadratic result as 32 bits at:

\[
\text{primary auxiliary value} + 0x3e.
\]

That store precedes the following test.

### 4. Sign-test exit

The adjusted value is multiplied by `0x10000`, and the product is tested for signed negativity.

If the product is negative, execution transfers directly to the shared return region:

- The lower return half is twice the low 32 bits returned by F-X05.
- The upper return half is the global return word minus `0x54`.

Both preceding 32-bit body stores have already occurred. No later 16-bit store, body call, byte load, derived-address store, or iteration-state replacement occurs.

Under the word model, this condition tests bit 15 of the adjusted value. It does not simply test whether the adjusted value itself is negative.

The supplied dependency map identifies the shared destination as **F-882dff43a182**. The represented control transfer is a jump to that region, not an established ordinary call.

### 5. Continuing body

If the sign test does not select the exit, the routine stores the low 16 bits of F-X05’s result at:

\[
\operatorname{upper32}(\text{F-X05 result}) + 0x3c.
\]

Thus, the same 64-bit result supplies both the address component and the stored data. No address validation is shown.

The routine then:

1. Calls **F-X06**, with no explicit arguments shown.
2. Calls **F-X07**, obtaining a 32-bit result.
3. Stores the 16-bit constant `0x0026` at F-X07’s result plus 2.

It computes a transformed value by multiplying the adjusted value by `0x10000000` and shifting the signed product right by nine positions.

Under the word model, this transformation depends only on the adjusted value’s low four bits. Its possible values range from `-0x00400000` through `0x00380000`, in increments of `0x00080000`.

The first-input state is then assigned the sum of:

- The transformed value.
- The adjusted value.

The routine reads the global XOR operand before calling **F-X08**. That call receives:

- Twice F-X07’s 32-bit result.
- The adjusted value multiplied by `0x10000`, minus `0x51`.

After F-X08 returns, the routine replaces the first-input state with its previously computed sum XOR the previously read global operand.

A write to the XOR global during F-X08 does not change the operand already selected for this iteration. This does not imply that F-X08 preserves other globals or register state.

The routine also adds `0x2f` to the transformed value. That offset-transformed value is used as a later call argument and stored value.

### 6. Final calls and iteration transition

The derived first-input value is assigned to represented stack storage at offset `0x130` from the displayed local stack origin.

The routine then reads one byte at:

\[
\text{derived first-input value} + 1.
\]

It calls **F-X09** with:

- The fixed location value `0x085734e8`.
- The loaded byte.
- The transformed value plus `0x2f`.
- The late-call auxiliary value.

The fixed location value is passed as an argument; this call site does not itself establish a memory read at that location. The precise promotion convention for the byte argument remains unresolved.

After F-X09 returns, the routine performs the following operations in this order:

1. Stores the transformed value plus `0x2f` as 32 bits at the derived first-input value plus `0x24`.
2. Assigns `0x08573405` to the represented link/dispatch state.
3. Calls **F-X10**, with no explicit arguments shown.
4. Replaces the second input with the iteration auxiliary value plus `0x30`.

The first-input state has already been replaced earlier in the iteration. It is not first replaced after F-X10.

The link/dispatch assignment likewise occurs **before F-X10**, not as part of a group of post-call updates. The representation does not independently establish what F-X10 does to actual register state or how the assigned value relates to that state after the call. Consequently, the visible assignment is established, but preservation across F-X10 is not.

Execution then repeats the loop. No iteration limit or independent termination condition is shown.

## Side effects

### Ordered direct accesses and state changes

Excluding accesses internal to dependencies, an iteration that reaches the continuing body exposes this sequence:

1. Read the global dependency argument after F-X01 and before F-X02.
2. Read the gate after F-X03.
3. Store the quadratic result as 32 bits at twice the second input plus `0x34`.
4. Call F-X04 and F-X05.
5. Store the quadratic result as 32 bits at the primary auxiliary value plus `0x3e`.
6. If the sign test permits continuation, store 16 bits at F-X05’s upper result half plus `0x3c`.
7. Call F-X06 and F-X07, then store `0x0026` as 16 bits at F-X07’s result plus 2.
8. Assign the pre-XOR sum to first-input state and read the global XOR operand.
9. Call F-X08, then replace first-input state with the XOR-derived value.
10. Assign that value to the represented stack slot and read one byte at that value plus 1.
11. Call F-X09, then store 32 bits at the derived first-input value plus `0x24`.
12. Assign `0x08573405` to represented link/dispatch state.
13. Call F-X10.
14. Replace the second input using the iteration auxiliary value plus `0x30`.

An exit reads the global return word during result construction. The gate-zero exit omits all body accesses; the sign-test exit stops after the second 32-bit body store.

### Aliasing and mutation

No non-aliasing guarantee is established. Computed addresses may overlap globals, stack storage, other computed destinations, or storage used by dependencies.

Therefore:

- Earlier stores may affect later reads.
- Dependencies may change globals before their next direct read.
- The final return word may reflect effects of earlier calls or stores.
- The gate need not remain constant between iterations.

The access sequence must not be replaced by an assumption that globals are immutable or that computed destinations belong to separate objects.

### Ownership, validity, and failures

No allocation, release, reference-count operation, or ownership transfer is explicitly identified.

Passing a stack address to F-X04 does not establish that the dependency owns the storage or may retain the address after its valid lifetime. The required storage size and dependency access pattern are unresolved.

Normal completion requires usable call targets and memory satisfying the accesses actually reached. The routine shows no checks for null addresses, mappings, alignment, read/write permissions, or address provenance.

No recovery or rollback is shown for invalid accesses or dependency failures. Earlier effects are not explicitly undone.

## Dependencies

These are opaque call-site IDs, except for the supplied identity of the shared return destination. Separate IDs do not establish distinct implementations.

| ID | Established interaction |
|---|---|
| F-X01 | Indirect target depends on link/dispatch state; receives three arguments. |
| F-X02 | Receives a global value read after F-X01. |
| F-X03 | Returns before the gate is read. |
| F-X04 | Receives a doubled input, a primary auxiliary value, and a stack address. |
| F-X05 | Returns a represented 64-bit value used for an exit candidate and an address/data split. |
| F-X06 | Called before F-X07; no explicit arguments shown. |
| F-X07 | Returns a 32-bit address component, also used in a doubled argument. |
| F-X08 | Runs after the XOR operand has been read and before the XOR is applied. |
| F-X09 | Receives a fixed location value, a loaded byte, an arithmetic value, and an auxiliary value. |
| F-X10 | Runs after the link/dispatch assignment and before second-input replacement. |
| F-882dff43a182 | Shared return destination reached by the conditional jump. |

No dependency is assumed to preserve globals, preserve actual registers, validate addresses, own passed storage, or release memory.

## Acceptance examples

These examples are conditional consequences of the static representation, not observed executions. Arithmetic examples use the inferred word model, and memory examples require valid storage.

1. **Zero gate still allows the initial calls.**  
   If the gate is zero when read after F-X03, F-X01 through F-X03 have already occurred. F-X04 does not occur. With a primary auxiliary value of `0x00000200`, the lower return half is 2.

2. **Negative gate-zero operand.**  
   With primary auxiliary bits `0xffffffff`, arithmetic right shift by nine yields `0xffffffff`; doubling gives lower-half bits `0xfffffffe`.

3. **Positive adjusted value causing an exit.**  
   An adjusted value of `0x00008000` produces signed-negative product bits `0x80000000` when multiplied by `0x10000`. The sign-test exit is selected even though the adjusted value is positive. Both preceding 32-bit body stores have occurred.

4. **Negative adjusted value not causing that exit.**  
   Adjusted bits `0xffff0000` produce zero when multiplied by `0x10000` under the word model. Negativity of the adjusted value alone does not select the exit.

5. **F-X05 result split.**  
   If F-X05 returns upper-half bits `0x00002000` and lower-half bits `0x12345678`, continuation stores `0x5678` at `0x0000203c`. If the sign-test exit is selected instead, that store is omitted and the lower return half is `0x2468acf0`.

6. **Cached XOR operand.**  
   If the XOR global is read as one value and F-X08 subsequently changes it, the derived first input uses the earlier value.

7. **Final transition ordering.**  
   A continuing iteration orders the derived-address 32-bit store before the `0x08573405` link/dispatch assignment, that assignment before F-X10, and F-X10 before second-input replacement. This example does not require F-X10 to preserve actual register state.

8. **Upper-half wrap.**  
   If the global return word is zero when read during final construction, the upper return half is `0xffffffac`.

## Uncertainties

The following remain **documented_only**:

- Exact calling convention, effective signature, and auxiliary-value producers.
- Auxiliary-register lifetimes across calls.
- How represented link/dispatch state corresponds to actual registers, including effects of F-X10 after the explicit assignment.
- Concrete signed-overflow and signed-shift behavior.
- Gate storage width and any volatile, atomic, or hardware-specific semantics.
- Byte-argument promotion.
- Address validity, alignment requirements, permissions, and endianness.
- Object layouts associated with computed addresses.
- Required size, permitted accesses, and lifetime constraints for stack storage passed to F-X04.
- Whether the represented stack-slot assignment is an observable executable memory store.
- Dependency implementations, aliasing, ownership behavior, and failure modes.
- Whether the shared return destination is also independently callable.
- Termination, concurrency, and reentrancy guarantees.

These unknowns prevent an unconditional safety or termination guarantee for arbitrary inputs.

## Evidence

The supplied static representation establishes the described call sequence, gate placement, access widths and offsets, conditional exits, and two-half return construction.

It also explicitly establishes the final continuing-iteration order: derived-address 32-bit store, link/dispatch assignment, F-X10, then second-input replacement. It does not independently establish preservation of actual register state across F-X10.

The supplied dependency map identifies F-882dff43a182 as the shared return destination. Other dependency IDs identify call sites without asserting implementation identity.

Wrapping arithmetic and arithmetic-right-shift behavior are qualified execution-model inferences where signed-expression semantics remain unresolved. No runtime tests, dependency execution, or memory observations have been performed.
