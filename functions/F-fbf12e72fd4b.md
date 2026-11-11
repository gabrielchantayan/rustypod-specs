<!-- Trusted provenance: F-fbf12e72fd4b; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-fbf12e72fd4b

## Purpose

F-fbf12e72fd4b repeatedly performs an ordered sequence of indirect calls, memory accesses, address calculations, and a software-breakpoint operation. The supplied representation contains **37 indirect calls per completed iteration**, followed by an unconditional loop back to the beginning. No normal return is represented.

Its application-level purpose is not established. The observed offsets and pointer-like values do not establish initialization, event processing, collection management, or memory ownership.

This contract describes the explicit local behavior. Each dependency may introduce additional effects or prevent continuation. Statements about later operations are conditional on earlier calls, accesses, and breakpoint handling completing.

## Interface

### Entry and exit

- No explicit parameters are displayed.
- No explicit return value or normal return path is displayed.
- Inputs also arrive through ambient machine state, globals, memory, and auxiliary dependency outputs.
- There is no represented local error result, exception handler, validation failure branch, or termination condition.

An inaccessible address, non-returning dependency, or breakpoint that does not resume can prevent further progress.

### Contract notation

The following names describe values, not recovered program symbols:

| Name | Meaning |
|---|---|
| **B** | Ambient address base used for several accesses and explicitly reduced during each iteration |
| **C** | Fixed 32-bit value `0x21C48960` |
| **G₁**, **G₂** | Two distinct global 32-bit storage locations |
| **P** | Value captured from **G₁** |
| **R** | Ambient register operand used by the late byte-store address |
| **L₁/H₁**, **L₂/H₂**, **L₃/H₃** | Low/high 32-bit halves of three 64-bit dependency results |

Other descriptive values are introduced where obtained. A captured value is distinguished from a later reload: for example, writes through **P** use the captured value, not a fresh read of **G₁**.

**B**, **R**, and the continuation-like input depend on incompletely specified machine-state conventions. The representation does not establish that arbitrary dependencies preserve the corresponding physical registers.

### Widths and layout

The explicit memory accesses use these widths:

- Byte: 8 bits.
- Halfword: 16 bits.
- Word, address operand, and global value: 32 bits.
- Paired dependency result: 64 bits, split numerically into two 32-bit halves.

Unsigned halfword reads are zero-extended when used in wider expressions. Byte and halfword stores retain only the low 8 or 16 bits of their value. A numeric low/high split does not establish memory byte order.

Offsets in this contract are byte offsets. An offset suggests neither a validated object boundary nor a proven structure type.

Arithmetic below describes 32-bit word data flow: left shifts, multiplication, and addition are expressed by their low 32-bit results where necessary; unsigned right shifts insert zeros. This is not a claim that signed expressions in the supplied representation define portable source-language overflow behavior. Actual addressing, alignment, overflow faults, and exception semantics remain environment-dependent.

### Dependency labels

**F-unresolved-01** through **F-unresolved-37** identify call occurrences within one iteration. These are opaque labels, not recovered function identities. Different occurrence labels need not denote different functions.

The first target depends on an incoming continuation-like value. The remaining target expressions are formed from displayed constants. Fixed numeric target expressions do not establish dependency signatures, ownership rules, or implementations.

A call with no displayed arguments is not proof that it consumes no ambient register inputs.

## Behavior

### 1. Enter the iteration and capture the first global

1. Establish an explicit byte value of zero.
2. Invoke **F-unresolved-01** with two explicit zero arguments. Its target is the incoming continuation-like value plus `0xFFE`.
3. Establish another explicit byte value of zero.
4. Invoke **F-unresolved-02**, then **F-unresolved-03**.
5. Read the 32-bit value of **G₁**, retaining it as **P**.
6. Invoke **F-unresolved-04**.
7. Invoke **F-unresolved-05**, obtaining the paired result **L₁/H₁**.
8. Invoke **F-unresolved-06** with:
   - First argument: **L₁**.
   - Second argument: the low 32 bits of **H₁ + L₁**.

The displayed second-argument expression also adds the explicit zero byte. It contributes zero in the represented data flow.

Immediately after **F-unresolved-06**, write its auxiliary 8-bit register result to **P + 0x11**. This auxiliary result is not established to be the call’s ordinary return value.

### 2. Perform narrow writes and extract a bit field

9. Invoke **F-unresolved-07**.
10. Write the explicit zero byte to **P + 0x15**.
11. Invoke **F-unresolved-08**, obtaining a 16-bit ordinary result and an auxiliary 32-bit address **Q**.
12. Write that 16-bit result to **Q + 0x0C**.
13. Invoke **F-unresolved-09**, obtaining **L₂/H₂**.
14. Write a zero halfword to **H₂ + 0x2C**.
15. Write **L₂ logically shifted right by 26 bits** as a word to **B + 0x26C**.

The last value is between 0 and 63 inclusive.

There is no visible reload of **G₁** before either write through **P**. Changes to **G₁** by intervening dependencies therefore do not themselves replace the captured address. They can still affect memory, accessibility, or other ambient state.

### 3. Update a captured destination and reach the breakpoint

16. Establish **C = 0x21C48960**.
17. Invoke **F-unresolved-10**.
18. Read a 32-bit pointer-like value **U** from **B + 0x320**.
19. Invoke **F-unresolved-11**, then **F-unresolved-12**.
20. Write the low 32 bits of **L₂ shifted left by 2** to **U + 0x54**.
21. Invoke, in order:
   - **F-unresolved-13**
   - **F-unresolved-14**
   - **F-unresolved-15**
   - **F-unresolved-16**
   - **F-unresolved-17**
22. Read an unsigned halfword **A** from **C + 0x34**, namely `0x21C48994`.
23. Invoke **F-unresolved-18**.
24. Read a word from **B + 0x280** and pass it to **F-unresolved-19**.
25. Retain the 32-bit result of that call as **J**.
26. Execute a software-breakpoint operation with immediate value `0x92`.

No subsequent operation is reached unless execution continues past the breakpoint. The representation supplies no local breakpoint handler.

### 4. Use narrow values and arithmetic constants as addresses

After breakpoint continuation, form:

- **I**: the low 32 bits of **J shifted left by 2**.
- **K**: **(C logically shifted right by 18) multiplied by 4**.

The displayed constant gives **K = 0x21C4**, or 8644.

Then:

27. Read a 32-bit pointer-like value **V** from **A + 0x44**.
28. Invoke **F-unresolved-20**.
29. Invoke **F-unresolved-21**, obtaining a 32-bit result **W**.
30. Read a word from **A + 0x04** and invoke **F-unresolved-22** with **W** and that word.
31. Read the 32-bit value of **G₂**, retaining it as **D**.
32. Invoke **F-unresolved-23**.
33. Write the low 16 bits of **D** to **I + D**.
34. Invoke **F-unresolved-24**.
35. Read a 32-bit pointer-like value **E** from **K + 0x54**.
36. Invoke **F-unresolved-25**, obtaining a 32-bit address **Z**.
37. Write the low 16 bits of **K** to **Z + 0x04**.
38. Invoke **F-unresolved-26**, then **F-unresolved-27**.
39. Read an unsigned halfword **N** from **E + 0x34**.
40. Invoke **F-unresolved-28**, obtaining **L₃/H₃**.

**A** is a zero-extended halfword used as an address base. The function does not load a full-width pointer from **C + 0x34**.

Similarly, **K** is calculated, not loaded as an object pointer. The read at **K + 0x54** is therefore a word read at `0x2218`. No preceding local validity check is represented.

### 5. Store a paired result and reduce the ambient base

41. Write **L₃** as a word to the current **B + 0x258**.
42. Read an unsigned halfword **M** from **H₃ + 0x14**.
43. Form:
   - **S**: the low 32 bits of **L₃ shifted left by 6**.
   - **T₄**: the low 32 bits of **H₃ multiplied by 4**.
44. Reduce **B** by `0xF4` bytes.
45. Form **N₄ = N multiplied by 4**.
46. Write **L₃** as a word to **N₄ + 0x74**.
47. Invoke **F-unresolved-29**.
48. Write zero-extended **M** as a word to **V + 0x54**.
49. Invoke **F-unresolved-30**.
50. Write zero-extended **M** as a word to **V + 0x64**.
51. Invoke **F-unresolved-31**, obtaining an auxiliary address result **X**.
52. Write the low 16 bits of **V** to **S + X**.

The write at **B + 0x258** uses the base before its explicit reduction. It is not a write at that offset from the reduced base.

The writes at **V + 0x54** and **V + 0x64** are separated by **F-unresolved-30**. They are not established to be atomic or free from intervening dependency effects.

### 6. Copy a byte, replace a field, and establish the continuation

53. Invoke **F-unresolved-32**, then **F-unresolved-33**.
54. Read a byte **Y** from **N + N₄**, equivalently **5 × N**.
55. Invoke **F-unresolved-34**.
56. Write **Y** to **M + R + 0x19**, using the represented ambient register operand **R**.
57. Invoke **F-unresolved-35**.
58. Write **T₄** as a word to **V + 0x44**.
59. Invoke **F-unresolved-36**.
60. Assign the continuation-like value **`0x08712031`**.
61. Read a word from **T₄ + 0x44**.
62. Invoke **F-unresolved-37** with that word.
63. If the final call returns, repeat the iteration unconditionally.

There is **no additional call between F-unresolved-36 and F-unresolved-37**. The continuation assignment precedes the final argument load, not merely the final call.

The representation carries the assigned continuation-like value toward the loop back edge. It does not independently establish that **F-unresolved-37** preserves the corresponding ambient machine state. If the value remains `0x08712031` when the next first-call target is formed, that target is `0x0871302F`.

The working byte used for **Y** is explicitly reset to zero at the beginning of the next iteration. The byte copy does not change the represented zero-store value for that iteration.

## Side effects

A completed iteration directly performs **15 stores**:

| Width | Count | Destinations |
|---|---:|---|
| Byte | 3 | **P + 0x11**, **P + 0x15**, **M + R + 0x19** |
| Halfword | 5 | **Q + 0x0C**, **H₂ + 0x2C**, **I + D**, **Z + 0x04**, **S + X** |
| Word | 7 | **B + 0x26C**, **U + 0x54**, pre-reduction **B + 0x258**, **N₄ + 0x74**, **V + 0x54**, **V + 0x64**, **V + 0x44** |

Other direct effects include the explicit base reduction, continuation-like assignment, breakpoint operation, and 37 indirect calls.

Destinations and sources may alias one another, globals, or dependency-accessed storage. No disjointness, synchronization, memory barrier, or atomic multi-access update is established. Access order therefore remains significant.

No local allocation, release, reference-count operation, or ownership transfer is established. A dependency may perform such actions, but its behavior is unknown. Passing or storing an address does not establish that the recipient acquires ownership.

The function shows no checks for null values, bounds, alignment, overflow, or accessibility. Continuing successfully requires each reached operation to be supported by the actual execution environment.

## Dependencies

The principal explicit data-producing or consuming call occurrences are:

| Dependency | Visible contract contribution |
|---|---|
| **F-unresolved-01** | Receives two zeros; target depends on continuation-like state |
| **F-unresolved-05** | Produces **L₁/H₁** |
| **F-unresolved-06** | Consumes derived arguments; supplies an auxiliary byte |
| **F-unresolved-08** | Supplies a halfword and auxiliary address **Q** |
| **F-unresolved-09** | Produces **L₂/H₂** |
| **F-unresolved-19** | Consumes a base-relative loaded word; produces **J** |
| **F-unresolved-21** | Produces **W** |
| **F-unresolved-22** | Receives **W** and a word read through **A** |
| **F-unresolved-25** | Produces destination address **Z** |
| **F-unresolved-28** | Produces **L₃/H₃** |
| **F-unresolved-31** | Supplies auxiliary address **X** |
| **F-unresolved-37** | Receives the word loaded after the continuation assignment |

All **37** occurrences remain potentially effectful. The absence of a listed result does not establish purity, absence of inputs, preservation of globals, preservation of ambient registers, memory ownership behavior, or normal return.

## Acceptance examples

These are conditional static examples, not runtime tests. They assume accessible addresses, returning dependencies, continuation past the breakpoint where required, and the stated 32-bit arithmetic interpretation.

1. **Paired-result addition narrows to a word.**  
   With **L₁ = 0xFFFFFFFF** and **H₁ = 2**, the explicit arguments to **F-unresolved-06** are `0xFFFFFFFF` and `1`.

2. **Different uses of one result retain different bits.**  
   With **L₂ = 0xFC000001**, the word at **B + 0x26C** receives `63`; the later word at **U + 0x54** receives `0xF0000004`.

3. **The halfword address base is unsigned.**  
   With **A = 0xFFFF**, the read at **A + 0x44** is at `0x10043`, not at an address obtained by sign extension.

4. **Only the stored value is narrowed.**  
   With **D = 0x12345678** and **I = 8**, the halfword destination is `0x12345680`, and its value is `0x5678`.

5. **The old base is used before reduction.**  
   With **B = 0x1000** at the **L₃** store, the destination is `0x1258`. The subsequent explicit reduction produces `0x0F0C`, absent another change.

6. **A halfword becomes a zero-extended word.**  
   With **M = 0xFFFF**, both writes through **V + 0x54** and **V + 0x64** supply `0x0000FFFF`. The intervening dependency may still affect the final memory contents.

7. **The byte source uses arithmetic on the narrow value.**  
   With **N = 7**, the byte source is address `35`. Its destination independently uses **M + R + 0x19**.

8. **Breakpoint non-continuation stops the local sequence.**  
   If breakpoint handling never resumes execution, no post-breakpoint access is required to occur.

9. **Continuation assignment precedes a failing final load.**  
   If **F-unresolved-36** returns but the read at **T₄ + 0x44** faults, the assignment of `0x08712031` has already been reached. **F-unresolved-37** has not been invoked.

10. **The next first target is conditional on retained state.**  
    If the final load succeeds, **F-unresolved-37** returns, and the assigned continuation-like state remains unchanged, the next **F-unresolved-01** target is `0x0871302F`. Without the preservation condition, that target is not guaranteed by the available calling-convention evidence.

## Uncertainties

The following remain **documented_only**:

- Dependency identities, complete signatures, calling conventions, and executable target validity.
- The ABI meaning and reliability of auxiliary register outputs.
- Whether **B** represents stack state or another address domain.
- Whether dependencies alter **B**, **R**, or continuation-like machine state.
- The application meaning, lifetime, and ownership of accessed storage.
- Endianness, alignment requirements, memory mapping, arithmetic fault behavior, and exception handling.
- Whether the breakpoint is intentional execution behavior, instrumentation, or a representation artifact.
- Whether low-address accesses are valid in the intended environment.
- Whether this entry point is independently callable with its displayed no-argument interface.

Repeated offset-based writes may suggest coordination of structured state. That is inferred intent, not an established application contract.

## Evidence

The supplied static representation establishes the explicit call-occurrence order, access widths, constants, paired-result extraction, narrow-value use, breakpoint operation, base reassignment, and unconditional repetition.

Direct enumeration yields **37 indirect calls**, **3 byte stores**, **5 halfword stores**, and **7 word stores** per completed iteration. The final sequence is **F-unresolved-35**, the word store at **V + 0x44**, **F-unresolved-36**, assignment of `0x08712031`, the word read at **T₄ + 0x44**, and **F-unresolved-37**.

The values **K = 0x21C4**, **K + 0x54 = 0x2218**, and **N + N₄ = 5 × N** follow arithmetically from the displayed expressions.

No runtime tests have been performed. No dependency implementation, memory map, or calling-convention documentation was supplied.
