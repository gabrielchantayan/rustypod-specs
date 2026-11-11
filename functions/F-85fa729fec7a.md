<!-- Trusted provenance: F-85fa729fec7a; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-85fa729fec7a

## Purpose

F-85fa729fec7a performs an ordered sequence of indirect dependency calls, four-byte stores, selector-byte reads, and byte stores, followed by an unconditional repeating call-and-store sequence.

No application-level purpose is established. The available evidence does not identify a protocol, data format, device operation, allocation operation, or successful completion condition. Its established contract is the visible ordering of effects and the arithmetic relationships between inputs, dependency results, addresses, and stored values.

If all dependencies return normally and all memory accesses complete, the recovered control flow remains in the final loop indefinitely. No normal return is visible.

This document distinguishes:

- **Established static evidence:** accesses, calls, arithmetic, and ordering visible in the supplied representation.
- **Conditional machine interpretation:** numerical consequences requiring 32-bit wrapping arithmetic or low-bit retention.
- **Inferred intent:** possible interpretations not established as behavior.
- **documented_only:** unresolved behavior for which no contract is available.

## Interface

### Explicit inputs

| Position | Visible width and interpretation | Established use |
|---|---|---|
| First | Four-byte uninterpreted value | Written to the initial word destination. |
| Second | Signed integer, represented as 32 bits | Multiplied by four; the scaled value determines a word destination, is passed to a dependency, and is later stored. |
| Third | Four-byte uninterpreted value | No explicit use is visible. |
| Fourth | Unsigned integer, represented as 32 bits | Logically shifted right by seven; the result determines a byte destination and a dependency argument. |

The third input’s lack of explicit use does not prove that indirect dependencies cannot observe its incoming machine representation.

### Implicit inputs

Execution also depends on:

- An incoming unsigned register value, logically shifted right by eight to form the **entry selector base**.
- An incoming return-link value, increased by 4,094 to form the first indirect-call target.
- Selector bytes read from computed addresses.
- Results returned by dependencies.
- Ambient state observed or changed by dependencies.

The four explicit inputs therefore do not determine execution by themselves.

### Outputs and completion

There is no visible return value or normal return path.

Directly visible outputs are:

- Two four-byte stores to one computed destination.
- Six byte stores before the loop.
- One byte store per completed loop iteration.
- Dependency invocations with the arguments described below.

Dependency effects are additional and unknown. A result unused by this routine does not imply that the producing call is effect-free.

### Widths and arithmetic

The representation explicitly distinguishes four-byte values, unsigned byte reads, and single-byte stores. Numerical descriptions use these conventions:

- A word is 32 bits.
- A byte is eight bits.
- Unsigned right shifts insert zeros.
- A byte store retains the least significant eight bits.
- Where explicitly stated, machine-level left shifts retain the least significant 32 bits.

The full architecture and ABI are not independently established. In particular, signed multiplication, signed left shifts, and signed address calculations cannot be assigned unconditional overflow behavior from the supplied representation alone. Results depending on wraparound are conditional, not portable signed-arithmetic guarantees.

The memory byte order of a four-byte store is unresolved.

## Behavior

### Dependency naming

**F-U01** through **F-U46** are opaque identifiers for call sites, assigned in first-invocation order. They are not verified function identities. Different call sites may resolve to the same function.

“No explicit arguments” means none are shown; it does not establish that a callee has no access to incoming registers, stack contents, or ambient state.

F-U37 requires a further distinction: its first invocation and later invocations use different target expressions. These are designated **F-T01** and **F-T02**, respectively.

### Initial word destination and first result

The second input is multiplied by four to produce the **scaled input**. The routine writes the first input as a four-byte value at:

**scaled input + 20**

This store precedes every dependency invocation. The entry selector base is then derived from the implicit register input by logical right shift by eight.

The initial dependency sequence is:

| Ordered calls | Explicit arguments and consumed results |
|---|---|
| F-U01 | Receives the scaled input. Its target is the incoming return-link value plus 4,094. |
| F-U02, F-U03, F-U04 | No explicit arguments. |
| F-U05 | No explicit arguments; its result is retained. |

After F-U05 returns, the scaled input is written as a four-byte value to the same destination as the first store.

Thus five calls separate the two word stores. Those calls may observe or modify the destination. The second store replaces the contents present at that time; it is not evidence that the first value remained unchanged throughout the interval.

Execution continues:

| Ordered calls | Explicit arguments and consumed results |
|---|---|
| F-U06 | Receives the F-U05 result shifted left by two. |
| F-U07, F-U08, F-U09 | No explicit arguments. |
| F-U10 | No explicit arguments; its result is retained as the **first extracted word**. |

### First two selector-controlled stores

Immediately after F-U10, the routine reads an unsigned byte from:

**entry selector base + 1**

It uses that byte as a numeric address component and stores the low byte of the first extracted word shifted left by two at:

**selector byte + 17**

The destination is not relative to the entry selector base. Because the selector is an unsigned byte, the destination is between numeric addresses 17 and 272 inclusive.

The next dependency sequence is:

| Ordered calls | Explicit arguments and consumed results |
|---|---|
| F-U11 | Receives three identical arguments, each the first extracted word shifted left by two. |
| F-U12 | No explicit arguments; its four-byte result is retained. |
| F-U13 | Receives the F-U12 result without a visible transformation. |
| F-U14, F-U15, F-U16 | No explicit arguments. |
| F-U17 | No explicit arguments; its result is retained. |

The entry selector byte is then read again. At the newly selected destination, the routine stores the low byte of:

**first extracted word masked to its low 30 bits, then logically shifted right by 20**

The intermediate can contain ten bits, but only eight survive the store. The stored byte is therefore original bits 20 through 27. Original bits 28 and 29 do not survive.

The second selector read is fresh. Intervening dependency effects may make its destination differ from that of the first selector-controlled store.

### Fourth-input destination and further entry-selector stores

Execution continues:

| Ordered calls | Explicit arguments and consumed results |
|---|---|
| F-U18 | Receives the F-U17 result shifted left by two. |
| F-U19, F-U20, F-U21 | No explicit arguments. |
| F-U22 | No explicit arguments; its result becomes the **second extracted word**. |

The second extracted word is shifted left by two. This full-width shifted value is retained for a later selector-address calculation.

Two ordered byte-store operations follow:

1. Store the low byte of that shifted value at **(fourth input logically shifted right by seven) + 28**.
2. Read the entry selector byte again, then store zero at **selector byte + 17**.

The direct byte store occurs before the selector read. If it aliases the selector source, the following read observes the effects of that store according to the applicable memory semantics. These operations must not be reordered in the behavioral contract.

The next sequence is:

| Ordered calls | Explicit arguments and consumed results |
|---|---|
| F-U23 | Receives the fourth input logically shifted right by seven. |
| F-U24 | No explicit arguments. |
| F-U25 | Receives the second extracted word masked to its low 30 bits, then logically shifted right by six. |
| F-U26, F-U27, F-U28 | No explicit arguments. |
| F-U29 | No explicit arguments; its result is retained. |

The F-U25 argument ranges from zero through 16,777,215.

After F-U29 returns, the routine reads the entry selector byte for the fourth time and stores zero at **selector byte + 17**.

### Second selector source and loop initialization

The next dependency sequence is:

| Ordered calls | Explicit arguments and consumed results |
|---|---|
| F-U30 | Receives the F-U29 result shifted left by two. |
| F-U31, F-U32, F-U33 | No explicit arguments. |
| F-U34 | No explicit arguments; its result is retained. |

The F-U34 result is masked to its low 30 bits and logically shifted right by 25. This produces the **initial loop token**, ranging from zero through 31.

A second selector address is derived from the retained full-width shifted second extracted word:

**retained shifted value × 1,024 + 1**

Under modulo-32-bit machine arithmetic, this equals the second extracted word shifted left by 12, retaining 32 bits, plus one. This equivalence does not resolve the signed-overflow semantics of the displayed multiplication.

The routine reads an unsigned byte from this second selector address and stores zero at **selector byte + 17**.

It then invokes:

| Ordered calls | Explicit arguments and consumed results |
|---|---|
| F-U35 | Receives the initial loop token shifted left by one: an even value from zero through 62. |
| F-U36 | No explicit arguments; its result becomes the first loop-carried argument. |

The target-base state used by the first F-U37 invocation is established before F-U36 is invoked.

### Repeating loop and target transition

Each iteration invokes these dependencies in order:

| Ordered calls | Explicit arguments and consumed results |
|---|---|
| F-U37 | Receives the current loop-carried argument and the current loop token logically shifted right by eight. |
| F-U38, F-U39, F-U40 | No explicit arguments. |
| F-U41 | No explicit arguments; its result is retained. |

After F-U41, the routine reads a fresh unsigned byte from the second selector address and stores zero at **selector byte + 17**.

The remainder of the iteration is:

| Ordered calls | Explicit arguments and consumed results |
|---|---|
| F-U42 | Receives the F-U41 result shifted left by two. |
| F-U43, F-U44, F-U45 | No explicit arguments. |
| F-U46 | No explicit arguments; its result determines both next-iteration arguments. |

The next loop-carried argument is the F-U46 result shifted left by two. The next loop token is that same result shifted left by 26, retained as an unsigned word.

The F-U37 target changes across the first iteration boundary:

| Invocation of F-U37 | Target selection visible in the representation |
|---|---|
| First | F-T01, whose target-base state was established before F-U36. |
| Every subsequent invocation | F-T02, whose target-base state was assigned near the end of the preceding iteration, before F-U46. |

The represented numeric target expressions evaluate to **0x08833787** for F-T01 and **0x088337C5** for F-T02. They differ by 62. These are distinct represented targets, not evidence of distinct verified function identities or independently validated executable entry points.

Consequently, the loop must not be modeled as repeatedly calling one invariant target at F-U37. Unknown dependency behavior makes this target transition significant even when arguments happen to be equal.

The first F-U37 second argument is zero because the initial token is at most 31. Under 32-bit low-bit retention, subsequent second arguments equal the preceding F-U46 result’s low six bits shifted left by 18. Their range is zero through 16,515,072, in multiples of 262,144.

The second selector address remains derived from F-U22. Neither the loop token nor F-U46’s result replaces it. Its byte contents are nevertheless reread each iteration.

There is no visible exit condition.

## Side effects

### Direct memory accesses

Before the loop, the routine performs:

- Two four-byte stores to **scaled input + 20**.
- Four unsigned byte reads from the entry selector source.
- One unsigned byte read from the second selector source.
- Six byte stores total: five selector-controlled stores and one fourth-input-directed store.

Each loop iteration performs one unsigned selector-byte read and one zero-byte store, in addition to dependency effects.

Selector reads are not a snapshot. Each controls its immediately associated destination using the value read at that point. Destinations may alias selector sources, earlier destinations, or the word destination. Any aliasing must respect the established access order.

No visible bounds, null, alignment, mapping, or writability checks precede these accesses.

### Preconditions and ownership

Reaching a particular operation requires all earlier calls to return and all earlier accesses to complete.

Successful access additionally requires the relevant computed address, width, alignment, and permissions to be valid in the actual execution environment. No memory-map contract is supplied, including for the low numeric selector-controlled destinations.

No allocation, release, ownership transfer, or lifetime guarantee is established. A dependency argument is not proven to be an address, owned object, handle, or scalar semantic value merely because of its width.

Dependencies may inspect or mutate ambient state, selector bytes, and destination memory. Their preservation of globals and their ownership or retention of arguments are unknown.

### Errors and partial effects

There is no visible local error return, rollback, recovery branch, or exception handler.

A fault or a dependency that does not return may prevent all later operations. Earlier completed effects are not visibly undone. The repeating loop is not established as an error-recovery mechanism.

## Dependencies

The argument/result relationships above are the only available dependency contracts.

| Opaque IDs | Role |
|---|---|
| F-U01–F-U10 | Initial word-store interval and production of the first extracted word. |
| F-U11–F-U17 | Repeated shifted arguments, result forwarding, and production of the next shifted argument. |
| F-U18–F-U22 | Production of the second extracted word. |
| F-U23–F-U29 | Fourth-input-derived and masked-word arguments. |
| F-U30–F-U34 | Production of the initial loop token. |
| F-U35–F-U36 | Loop initialization. |
| F-U37–F-U46 | Repeating call sequence and loop-carried result production. |
| F-T01, F-T02 | First and recurrent target selections at F-U37. |

Most indirect targets use fixed address-like bases plus offsets. This does not establish a shared dispatcher, common callee, or operating-system interface. All unprovided dependency semantics remain **documented_only**.

## Acceptance examples

These are conditional criteria derived from static evidence, not executed tests. They assume valid accesses and normal dependency returns unless stated otherwise.

1. **Initial overwrite ordering.** With second input 3, the scaled input is 12 and the word destination is numeric address 32. The first input is stored there before F-U01. F-U01 receives 12. After F-U05 returns, the routine stores the four-byte value 12 at address 32.

2. **Selector value at the read.** If the selector byte equals 5 when the first selector read occurs after F-U10, the first selected destination is address 22. If it equals 9 at the next selector read after F-U17, that store targets address 26. A value of 5 only at routine entry is insufficient: F-U01 through F-U10 may change it.

3. **Low-byte truncation.** If the first extracted word is 255, its first stored byte is 252. Each of F-U11’s three explicit arguments is 1,020.

4. **High-field truncation.** If the first extracted word is 0x3FF00000, masking and shifting produces 1,023, but the byte store writes 255.

5. **Fourth-input addressing.** With fourth input 255, its shifted value is 1. The direct byte store targets address 29, and F-U23 receives 1.

6. **Store-before-read aliasing.** If the fourth-input-directed destination equals the entry selector source, and ordinary byte-memory semantics apply, the following selector read uses the newly stored byte. It does not use a value cached before the direct store.

7. **Initial loop arguments and target.** For every initial token from zero through 31, F-U35 receives twice that token and the first F-U37 second argument is zero. That F-U37 invocation uses F-T01.

8. **Loop transition.** If F-U46 returns 3, the next F-U37 receives 12 and 786,432 under 32-bit low-bit retention. It uses F-T02, not F-T01.

9. **No successful return.** If every access and call continues normally, completing one iteration begins another. No success return occurs in the recovered control flow.

## Uncertainties

- **Recovery reliability — documented_only.** The supplied representation reports invalid instruction data and removal of unreachable regions. Actual executable control flow has not been independently validated.
- **Architecture and ABI — documented_only.** Complete calling convention, register preservation, instruction-mode handling, pointer semantics, and endianness remain unresolved.
- **Overflow — documented_only.** Overflowing signed scaling, shifts, and address arithmetic require machine-level clarification. Conditional wrapping examples do not settle them.
- **Target validity — documented_only.** F-T01 and F-T02 are distinct represented target expressions; executable mapping and target interpretation are unverified.
- **Memory semantics — documented_only.** The evidence does not establish ordinary RAM, device-backed storage, access atomicity, or concurrent visibility.
- **Dependency behavior — documented_only.** Purity, global preservation, ownership, argument retention, error conventions, and guaranteed return are unknown.
- **Higher-level purpose — inferred intent only.** Repeated shifts, masks, and byte stores suggest bit-field manipulation, but no format or protocol is established.
- **Termination — static evidence only.** No local exit is visible. Faults, external interruption, dependency behavior, and recovery uncertainty preclude a claim of unconditional physical nontermination.

## Evidence

This contract is based solely on the supplied static representation. No runtime tests, executable validation, or independent implementation inspection have been performed.

Established claims concern visible widths, access order, explicit arguments, consumed results, target selection, and recovered control flow. Wrapping arithmetic and related numerical consequences are explicitly conditional.

Opaque dependency identifiers label unresolved call sites and target selections, not independently recovered function identities. No snapshot hash or independently verified provenance is asserted.
