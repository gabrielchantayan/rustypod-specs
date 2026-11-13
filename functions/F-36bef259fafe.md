<!-- Trusted provenance: F-36bef259fafe; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-36bef259fafe

## Purpose

F-36bef259fafe specifies a recovered execution path whose visible behavior consists of external invocations, coprocessor transfers, and a final attempt to write the 16-bit value **0x00CD** to a calculated address.

No application-level purpose is established. The available evidence does not justify describing this path as initialization, decoding, device configuration, or resource management.

The representation warns of invalid instruction data and ends at a truncated-control-flow marker. Accordingly, this contract distinguishes:

- **Established static evidence:** operations and value relationships displayed in the supplied representation.
- **Derived static consequences:** conclusions obtained from those relationships under explicitly stated widths.
- **Documented-only unknowns:** actual executability, dependency behavior, hardware semantics, and termination behavior.

All execution descriptions are conditional on the recovered path being accurate and every preceding operation permitting continuation.

## Interface

### Explicit inputs

The displayed interface has three 32-bit arguments.

| Input | Visible use |
|---|---|
| First argument | No explicit use in the recovered body |
| Second argument | Second displayed operand of F-I001 |
| Third argument | Third displayed operand of F-I001 |

Absence of visible use does not establish that changing the first argument is unobservable to unknown dependencies through the actual machine calling convention.

No explicit argument is established as a buffer, object, length, ownership token, or validated address.

### Ambient inputs

The path additionally depends on:

- A 32-bit incoming value used to construct F-I001’s first operand.
- Incoming link-related state used to calculate F-I001’s target.
- Eight symbolic incoming selector values used by coprocessor operations.
- Three residual register observations supplied to F-I014, F-I026, and F-I039.
- An execution environment capable of resolving and executing the computed invocation targets.

The residual observations are independent unknowns. They are not established as zero, equal to one another, or ordinary declared function results.

These dependencies do not establish a complete ABI. In particular, the representation does not prove how retained values survive intervening calls.

### Results and termination

There is no displayed normal return or ordinary return value. The last visible memory operation is followed by an invalid-instruction termination marker.

The contract does not promise:

- That the final store succeeds.
- That execution reaches the marker.
- That the marker represents a deliberate executable instruction.
- A particular exception, signal, error code, or recovery action.

Some dependencies expose apparent 64-bit results. Here, **low word** means bits 0–31 and **high word** means bits 32–63. This is a value decomposition, not a statement about memory byte order or the physical return convention.

### Width conventions

Unless explicitly identified as 64-bit, arithmetic payloads are treated as 32-bit bit patterns.

A left shift retains only the low 32 result bits. A logical right shift introduces zeros. An arithmetic right shift replicates the sign bit.

These conventions describe the recovered bit relationships. They do not assert that signed-looking expressions in the supplied representation have portable language-level semantics.

## Behavior

### External interaction order

F-I001 through F-I087 are opaque identifiers assigned to the **87 indirect invocation sites in execution order**. They identify sites, not necessarily distinct target functions.

Subject to continuation:

- The indirect sites occur in ascending identifier order.
- F-0e02e4c13579 occurs between F-I013 and F-I014.
- F-3de949054bdd occurs between F-I045 and F-I046.
- Coprocessor transfers occur at the boundaries specified under **Side effects**.
- The terminal store follows F-I087.
- No conditional branch, retry, validation failure path, or normal-return alternative is displayed.

Unless operands or retained results are specified below, an indirect site has no displayed arguments and no subsequently consumed result. This does not establish a zero-argument machine interface or absence of side effects.

### Entry dispatch and initial scalar

F-I001 has four displayed operands. Its first operand is formed by logically shifting the incoming ambient value right by three, then shifting the resulting 32-bit pattern left by sixteen. Its remaining operands are the second explicit argument, the third explicit argument, and zero.

Only original ambient bits 3–18 can contribute to that first operand. F-I001’s target depends on incoming link-related state; no target-validity check is displayed.

F-I002 supplies the first coprocessor payload.

F-I007 supplies a two-word result. Its low word is transferred to a coprocessor and also becomes F-I008’s first operand. F-I008’s second operand is F-I007’s high word shifted left by twenty.

F-I009 receives F-I008’s low word, its high word, and that high word shifted left by twelve.

F-I013 supplies the **initial scalar**. F-0e02e4c13579 receives this scalar unchanged. A separately retained pattern equals the scalar shifted left by eleven. No result from the direct dependency is visibly checked.

### Zero-derived operands and early result forwarding

F-I014 receives zero, an unresolved residual register observation, and zero.

F-I019 supplies a two-word result. F-I020 receives its low word shifted left by four, its high word unchanged, and the initial scalar shifted left by twenty-eight. Only the initial scalar’s low four bits can contribute to that third operand.

F-I021 receives F-I020’s two result words and a zero third operand. F-I022 consumes F-I021’s 32-bit result.

F-I024 receives two zeros; F-I025 consumes its result.

F-I026 has four displayed operands:

- Zero derived by further shifting the saved initial-scalar pattern.
- A second unresolved residual register observation.
- Zero.
- Zero.

The first operand is zero for every 32-bit initial scalar: the saved pattern already has eleven low zero bits, and its further left shift by thirty-one discards every possible set bit.

F-I026’s result is transferred to a coprocessor before being supplied to F-I027.

### Intermediate forwarding and middle-scalar transformation

The following producer-consumer relationships are visible; they imply no application-level interpretation of the values:

| Producer | Subsequent consumption |
|---|---|
| F-I028 | Result supplied to F-I029; F-I029’s result supplied to F-I030 |
| F-I034 | Result supplied to F-I035 |
| F-I044 | Result supplied to F-I045 |
| F-I046 | Result supplied to F-I047 |
| F-I048 | Result supplied to F-I049 |

F-I036 receives zero. F-I039 receives zero, the third unresolved residual register observation, and zero.

F-I042 supplies the **middle scalar**. F-I043 receives the middle scalar unchanged, zero, and the middle scalar shifted left by five.

F-3de949054bdd occurs after F-I045 and before F-I046. No displayed result is inspected.

F-I050 receives zero and a sign-extended reconstruction of the middle scalar’s low 27 bits: the middle scalar is first shifted left by five within 32 bits, then arithmetically shifted right by five. The reconstruction’s sign is determined by original bit 26, not original bit 31. F-I051 consumes F-I050’s result.

### Upper-bit pattern and repeated coprocessor payloads

F-I052 supplies a two-word result. Its low word becomes a coprocessor payload; its high word shifted left by one is supplied to F-I053.

F-I054 supplies one 32-bit result used in two consecutive coprocessor transfers. The payload is identical, but the selector pair differs.

F-I055 supplies the value from which the **upper-bit pattern** is formed by shifting left by twenty-seven. Only the original low five bits survive, occupying positions 27–31.

F-I056 receives zero, zero, this pattern, and zero.

F-I062 supplies a later coprocessor payload.

A subsequent transfer and F-I066’s first operand use the upper-bit pattern shifted left by twenty-four. Both are necessarily zero. F-I066’s second operand is also zero.

F-I068 supplies another coprocessor payload.

### Derived scalar and later forwarding

F-I070 receives three zeros and supplies the **derived scalar**.

That scalar shifted left by one becomes a coprocessor payload. F-I071 receives:

- The derived scalar shifted left by one.
- Zero, derived by selecting the upper-bit pattern’s low 23 bits and shifting them left by four.
- The derived scalar shifted left by four.
- Zero.

Two zero-payload coprocessor transfers follow F-I073.

A retained field selects the low two bits of the derived scalar’s left-by-four pattern and moves them upward by sixteen positions. This field is necessarily zero.

F-I074 receives zero. A coprocessor transfer after that call uses the derived scalar shifted left by twelve.

F-I075 receives the derived scalar shifted left by twelve, the derived scalar shifted left by sixteen, and F-I074’s result shifted left by nine.

F-I076 supplies a two-word result. F-I077 receives its low word, its high word, and its low word shifted left by one.

F-I078 receives zero derived from the retained field. Its result is supplied to F-I079, whose result is supplied to F-I080.

F-I082 supplies a two-word result. F-I083 receives those two words and the high word shifted left by eleven.

### Terminal address and write

F-I083 supplies an apparent 64-bit result. Logically shifting that result right by forty-one produces a **23-bit terminal seed**, in the range 0 through 8,388,607.

F-I084 receives F-I083’s low word. F-I085 supplies F-I086’s first operand; F-I086’s second operand is zero derived from the retained field. F-I087 follows.

The terminal destination depends only on F-I083’s high word:

- Let \(q\) be that high word logically shifted right by twenty.
- The destination address is \(q+\lfloor q/4\rfloor\).

This is equivalent to shifting the terminal seed right by eleven and adding one quarter of that shifted value. The seed is nonnegative under either signed or unsigned 32-bit interpretation, so the displayed signed-looking right shift does not introduce sign bits here.

Consequences:

- \(q\) ranges from 0 through 4095.
- Destination addresses range from 0 through **5118**.
- The arithmetic cannot overflow 32 bits.
- Reachable addresses have remainder 0, 1, 2, or 3 when divided by five; addresses with remainder 4 are skipped.
- F-I083’s low word does not affect the destination.
- Zero and odd addresses are possible.

The body attempts a **16-bit write of 0x00CD** directly to this address. No base object, bounds check, null check, alignment adjustment, or permission check is displayed.

The invalid-instruction termination marker follows the store in the representation.

## Side effects

### Calls and state visibility

Every external invocation is potentially observable. Unknown dependencies may read or modify memory, globals, devices, register state, or resource ownership.

Ascending site order does not imply preservation of shared state between calls. A dependency may observe changes made by any earlier operation. Ignored results do not make calls optional or side-effect-free.

### Coprocessor transfers

There are **thirteen** displayed coprocessor transfers.

The selector labels below are contract notation, not architectural register names. **T** denotes the common symbolic second selector. **B–H** denote seven other symbolic incoming selector values. Different labels identify different input roles, not necessarily numerically different values.

| Order | Position | Operation triplet | Payload | Selector pair |
|---|---|---|---|---|
| 1 | After F-I002, before F-I003 | (0, 5, 7) | F-I002 result | T, T |
| 2 | After F-I007, before F-I008 | (0, 4, 7) | F-I007 low word | B, T |
| 3 | After F-I026, before F-I027 | (0, 6, 7) | F-I026 result | C, T |
| 4 | After F-I052, before F-I053 | (0, 6, 7) | F-I052 low word | C, T |
| 5 | After F-I054, before transfer 6 | (0, 3, 7) | F-I054 result | D, T |
| 6 | After transfer 5, before F-I055 | (0, 3, 7) | Same F-I054 result | T, T |
| 7 | After F-I062, before F-I063 | (0, 1, 7) | F-I062 result | E, T |
| 8 | After F-I065, before F-I066 | (0, 5, 7) | Zero | F, T |
| 9 | After F-I068, before F-I069 | (0, 6, 7) | F-I068 result | G, T |
| 10 | After F-I070, before F-I071 | (0, 5, 7) | Derived scalar shifted left by one | H, T |
| 11 | After F-I073, before transfer 12 | (0, 7, 7) | Zero | G, T |
| 12 | After transfer 11, before F-I074 | (0, 0, 7) | Zero | T, T |
| 13 | After F-I074, before F-I075 | (0, 2, 7) | Derived scalar shifted left by twelve | G, T |

Repeated symbolic selectors express recovered dataflow, not a proven register-preservation guarantee across unknown callees.

Privilege requirements, device effects, synchronization semantics, and fault behavior remain unknown.

### Memory layout and ownership

The final halfword store is the only explicit ordinary data-memory access shown outside dependency bodies. No explicit buffer read, allocation, release, or object layout is established.

The store has a 16-bit payload. Its byte order, atomicity, alignment requirements, and behavior on inaccessible memory are unknown. A successful conventional two-byte store would occupy the destination and the following byte; the representation does not establish how a faulting store affects either byte.

No ownership obligation can be inferred from this body. In particular, it is not established that dependencies preserve, acquire, retain, or release any resource.

## Dependencies

- **F-0e02e4c13579:** receives the initial scalar as one displayed 32-bit argument.
- **F-3de949054bdd:** has no displayed arguments and no inspected result.
- **F-I001–F-I087:** unresolved indirect invocation sites, numbered by visible execution order.

The indirect sites use computed code references. Target identities, aliasing between sites, complete signatures, calling conventions, and successful return behavior are not established.

A displayed operand list is not a complete account of ambient inputs. Conversely, a locally retained value in the representation is not proof that a particular physical register survives an external call.

## Acceptance examples

These are conditional static examples, not executed tests.

1. **Argument forwarding.** If F-I001 is reached, its second and third displayed operands equal the second and third explicit arguments. The first explicit argument has no established direct use.

2. **Ambient-bit selection.** An ambient value of 8 produces F-I001’s first operand 0x00010000. A value of 7 produces zero.

3. **Initial-scalar truncation.** An initial scalar of 1 makes F-I020’s third operand 0x10000000. An initial scalar of 16 makes that operand zero. F-I026’s first operand is zero in both cases.

4. **Lossy signed reconstruction.** A middle scalar of 0x08000000 produces zero as F-I050’s second operand. A middle scalar of 0x04000000 produces 0xFC000000 because original bit 26 becomes the reconstructed sign bit.

5. **Upper-bit pattern.** If F-I055 returns 1, F-I056’s third operand is 0x08000000. Transfer 8 and F-I066 still receive their specified zero payloads.

6. **Distinct repeated transfers.** If F-I054 returns 0x12345678, transfers 5 and 6 both carry that value. Their selector pairs remain D,T and T,T respectively; equality of payloads does not collapse the two operations.

7. **Null destination.** If F-I083’s high word is below 0x00100000, the destination is zero. If all preceding operations continue, a 16-bit write of 0x00CD is attempted there without a displayed guard.

8. **Odd destination.** A high word of 0x00100000 yields \(q=1\) and destination 1. No visible alignment correction occurs.

9. **Maximum destination.** A high word of 0xFFFFFFFF yields \(q=4095\) and destination 5118. Changing only the low word leaves this calculation unchanged.

10. **Interrupted execution.** A nonreturning dependency or earlier fault prevents any guarantee about later transfers, the terminal store, or the termination marker.

## Uncertainties

The following remain **documented_only**:

- Intended application behavior and valid entry conditions.
- Accuracy of instruction recovery and function boundaries.
- Actual indirect targets and whether all displayed operations are executable.
- Complete ABI, apparent 64-bit returns, and residual register provenance.
- Preservation of retained values across dependencies.
- Coprocessor interpretation, privilege, ordering guarantees, and exceptions.
- Memory mapping, byte order, alignment rules, and store-fault behavior.
- Dependency effects on globals, memory, and ownership.
- Whether the final marker represents an executable invalid instruction, unreachable material, or recovery truncation.

No interpretation such as obfuscation, corruption, maliciousness, or initialization is adopted as a behavioral requirement.

## Evidence

**Established static evidence:** three displayed 32-bit explicit arguments; 87 indirect invocation sites; two identified direct dependencies; thirteen coprocessor transfers; apparent 32-bit and 64-bit value flows; one explicit 16-bit store; and a terminal invalid-instruction marker.

**Derived static consequences:** forced-zero operands, the sign-extension behavior of the middle-scalar reconstruction, the 23-bit terminal seed, and the destination range and skipped-address pattern.

**Inferred intent:** none adopted.

**Verification boundary:** static analysis of the supplied representation only. No runtime tests, dependency inspection, instruction validation, or execution-environment checks have been performed.
