<!-- Trusted provenance: F-967c0ddb23df; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-967c0ddb23df

## Purpose

F-967c0ddb23df performs an unconditional sequence of memory accesses, then selects additional writes and dependency calls using an implicit 32-bit branch value and an entry carry condition.

No application-level purpose is established. The supplied evidence does not justify identifying the function as an allocator, constructor, parser, or device operation. Several effective addresses come directly from zero-extended bytes or halfwords rather than from explicit pointer arguments.

This contract distinguishes:

- **Established static evidence:** represented access widths, value dependencies, ordering, branch conditions, and call-target calculations.
- **Inferred intent:** descriptive labels such as “base” and “selector.” These do not establish data structures or application meaning.
- **documented_only unknowns:** behavior not determined by the supplied representation, including dependency semantics, machine-level arithmetic details, and incomplete terminal control flow.

No runtime tests have been performed.

## Interface

### Explicit inputs

The represented interface has four inputs and no explicit return value.

| Input | Width | Established use |
|---|---:|---|
| First argument | 32 bits | No explicit use in the visible body. |
| Second argument | 32 bits | No explicit use in the visible body. |
| Third argument | 32 bits | Address component for initial byte writes; later stored as a word on the carry-clear path. |
| Fourth argument | 32 bits | Address component for byte and halfword reads; later stored as a word on the carry-clear path. |

The third and fourth arguments are represented as signed 32-bit quantities, but their visible uses are address arithmetic and raw word storage. No range, alignment, null, or mapping validation is shown.

The absence of explicit uses for the first two arguments does not establish their irrelevance to dependencies or an unknown calling convention.

### Additional entry state

Execution also depends on:

- A fixed-location 32-bit value, referenced multiple times during the unconditional prefix.
- An implicit entry byte.
- An implicit auxiliary base.
- An implicit 32-bit branch value.
- An entry carry condition.
- An entry link value used in certain call-target calculations.

The origins, initialization requirements, and ABI roles of this state are documented_only.

The **captured seed** below means the value obtained by the first fixed-location reference. It does not automatically mean the values obtained by later references to that location.

### Outputs and completion

Observable outputs are memory writes, dependency invocations, and any effects of those dependencies. No explicit status, ownership token, or result object is returned.

Completion differs by path:

- The nonzero, carry-clear path reaches an unrecovered indirect transfer represented as a call, followed by a return.
- The zero-value and nonzero, carry-set paths reach a final indirect call, followed by an invalid-instruction termination marker.

A successful return is not established for all paths. No rollback, recoverable error report, or cleanup protocol is visible.

## Behavior

### Widths, addresses, and evaluation limits

The represented memory operations use 8-bit bytes, 16-bit halfwords, and 32-bit words. Two dependencies expose represented 64-bit results, subsequently divided into low and high 32-bit portions.

The selector byte is unsigned and zero-extended for address formation. Both halfwords used as derived bases are unsigned and zero-extended. The halfword passed to F-U07 is represented only as a 16-bit quantity; its argument-extension convention is not established.

Address operands and intermediate integer values are represented in a 32-bit environment. Exact machine-level wrapping, signed-overflow behavior, alignment requirements, endianness, and access-fault behavior are documented_only. Expressions involving signed addition, doubling, or negation must not be treated as unbounded mathematical arithmetic.

The fixed call-target calculations listed under **Dependencies** involve small positive additions that fit within 32 bits. Entry-link-dependent target calculations have no established overflow behavior.

### Unconditional prefix

The following sequence precedes either branch decision:

1. Read the fixed-location value into the captured seed.
2. Perform a 32-bit word store whose address is twice a further reference to the fixed-location value, and whose stored value is another reference to that same location.
3. Read a byte at twice the captured seed.
4. Store the entry byte at the third argument plus the captured seed.
5. Store the byte obtained in step 3 at the third argument plus 5.
6. Read an unsigned byte at the fourth argument plus the captured seed. Retain it as the **selector byte**.
7. Read an unsigned halfword at the fourth argument plus 42. Zero-extend it as the **initial derived base**.
8. Store the branch value as a 32-bit word at the initial derived base plus 100.

Step 2 contains separate represented references for the store address and store value. The supplied expression does not establish their evaluation order relative to each other. Both belong after the initial capture and before completion of that store.

This is a distinction between represented value references, not a guarantee of three distinct physical memory loads. Optimization, storage qualifiers, and actual access semantics are not established.

Consequently, step 3 is not unconditionally a read from the address written in step 2. That equivalence requires the address-producing fixed-location reference to equal the captured seed. The stronger description “write the seed at twice the seed, then read its first byte” additionally requires the data-producing reference to equal the captured seed. Stable ordinary storage with no intervening mutation is a sufficient assumption, not an established property.

Even under that assumption, the byte read depends on endianness and memory semantics.

Aliasing is otherwise unrestricted. Either initial byte write can affect the later selector or initial-base read if addresses overlap. If the destinations in steps 4 and 5 coincide, step 5 is the later store.

### Zero branch value

If the branch value is zero:

- The unconditional prefix, including the word store of zero at the initial derived base plus 100, has already occurred.
- No auxiliary-base write is performed by the visible body.
- The carry condition does not select additional work.
- F-U10 is invoked at the entry link value plus 0x6BE.
- The visible path then reaches the invalid-instruction termination marker.

No restoration of earlier writes is shown.

### Nonzero branch value: common write

If the branch value is nonzero, the function stores the low 8 bits of the captured seed at the auxiliary base plus 1.

The entry carry condition then selects the remaining path. It is an implicit state input, not a comparison involving an explicit argument.

### Nonzero branch value with clear entry carry

The visible sequence is:

1. Invoke F-U01 using the entry link value plus 0xECC.
2. Store the fourth argument as a 32-bit word at the initial derived base plus 52.
3. Invoke F-U02.
4. Store the third argument as a 32-bit word at the initial derived base plus 36.
5. Invoke F-U03.
6. Store the represented 32-bit return-side state exposed after F-U03 at the initial derived base plus 20.
7. Invoke F-U04.
8. Invoke F-U05.
9. Read an unsigned halfword at the fourth argument plus an offset exposed by the preceding call’s return-side state. Zero-extend it as the **replacement derived base**.
10. Invoke F-U06 and obtain its represented 64-bit result.
11. Store the selector byte at the replacement derived base plus the selector byte.
12. Read a 16-bit value at the address formed by adding the low and high 32-bit portions of F-U06’s result.
13. Invoke F-U07 with four represented inputs, in order: the low portion, the high portion, the negation of the low portion, and the halfword just read.
14. Invoke F-U08.
15. Invoke F-U09.
16. Reach F-U11 through the unrecovered indirect transfer.

The representation places a return after F-U11. It does not establish that F-U11 returns, or that the transfer is an ordinary call rather than incompletely recovered control flow.

F-U03 and F-U04 have the same calculated target. They remain two invocations separated by the word store at offset 20. This equality does not establish that either invocation has the same effect as the other.

The selector used in step 11 is the byte captured during the unconditional prefix. It is both the stored value and a zero-extended address offset; there is no new selector read.

The earlier stores at offsets 52, 36, and 20 use the initial derived base. The replacement base is used for the later selector-byte store.

The low and high result portions are represented as signed 32-bit quantities for the address addition and explicit arguments. Overflow in their sum is unresolved. Negating the minimum signed 32-bit value is likewise unresolved at the source-expression level; modular machine negation would preserve its bit pattern, but that behavior is not independently established.

### Nonzero branch value with set entry carry

The visible sequence is:

1. Read a byte at the branch value plus 5.
2. Store that byte at the auxiliary base plus 5.
3. Invoke F-fa0c227e09d7.
4. Read a 32-bit word at the selector byte plus 84. Retain the loaded word as a further base.
5. Invoke F-U12 and obtain its represented 64-bit result.
6. Read a 32-bit word at the further base plus 36.
7. Invoke F-U13 with the low 32-bit result portion, the high 32-bit result portion, and the word just read, in that order.
8. Store the branch value as a 32-bit word at the initial derived base plus 116.
9. Establish the link-derived target basis as 0x08572D09 and invoke F-U14.
10. Invoke F-U10 at the resulting fixed target, 0x085733C7.
11. Reach the invalid-instruction termination marker.

The word-read address in step 4 is directly selector-derived. It is not relative to the fourth argument. Its numerical range is 84 through 339 inclusive.

The further-base value is captured before F-U12, but the word at that base plus 36 is read after F-U12. That word therefore cannot be assumed to have retained its earlier contents across the call.

F-U13 and F-U14 have the same calculated target. The branch-value store at offset 116 occurs between these invocations.

Unlike the zero-value path, F-U10’s target here does not depend on the entry link value. The visible assignment fixes its calculation.

## Side effects

Visible writes occur in the stated order. No transaction, synchronization, atomicity, rollback, or error-cleanup mechanism is represented.

Potentially accessed regions include:

- The fixed data location and seed-derived addresses.
- Addresses formed from the third and fourth arguments.
- Low-address regions selected by zero-extended halfwords.
- The auxiliary base and branch-value-derived addresses.
- A low-address word selected by the selector byte.
- Addresses derived from dependency results.

No allocation, release, or ownership-transfer operation is identified. The function does not visibly acquire ownership of either explicit address argument or any derived address.

Every dependency may have additional effects not described here. Its memory accesses, global mutations, retention or release of memory, exceptional behavior, and implicit state changes are documented_only. In particular, no dependency is assumed to preserve globals. A location written before a call is not guaranteed to retain that value afterward.

The represented value flow identifies values retained for later use; it does not independently verify register preservation or the executable ABI.

## Dependencies

The IDs below identify invocation roles. Distinct role IDs do not imply distinct targets. Numeric expressions describe target calculations, not known dependency semantics.

| Opaque ID | Established target calculation and role |
|---|---|
| F-fa0c227e09d7 | Explicitly identified dependency on the carry-set path, after the auxiliary byte transfer. |
| F-U01 | Entry link plus 0xECC; first carry-clear call. |
| F-U02 | 0x08572DD9 plus 0xEC6 = **0x08573C9F**. |
| F-U03 | 0x08572DDD plus 0xEC6 = **0x08573CA3**; precedes the offset-20 store. |
| F-U04 | 0x08572DE1 plus 0xEC2 = **0x08573CA3**; same target as F-U03. |
| F-U05 | 0x08572DE5 plus 0xEBC = **0x08573CA1**. |
| F-U06 | 0x08572DE9 plus 0xEB4 = **0x08573C9D**; exposes a represented 64-bit result. |
| F-U07 | Clear the low two bits of 0x08572DED, then add 0x288: **0x08573074**; receives four represented inputs. |
| F-U08 | 0x08572EAD plus 0xE88 = **0x08573D35**. |
| F-U09 | 0x08572EB1 plus 0xE8A = **0x08573D3B**. |
| F-U10 | Zero-value path: entry link plus 0x6BE. Carry-set path: 0x08572D09 plus 0x6BE = **0x085733C7**. |
| F-U11 | Unrecovered indirect terminal transfer; target calculation unresolved. |
| F-U12 | 0x08572CFD plus 0xEDA = **0x08573BD7**; exposes a represented 64-bit result. |
| F-U13 | 0x08572D01 plus 0xED2 = **0x08573BD3**; receives three represented inputs. |
| F-U14 | 0x08572D05 plus 0xECE = **0x08573BD3**; same target as F-U13. |

The equalities above are established arithmetic relationships. They do not establish symbol identities, instruction-set target interpretation, valid executable mappings, or callee behavior. Other roles could also coincide for particular entry-link values; no general distinctness claim is made.

A call without represented explicit arguments is not evidence of an argument-free ABI. Implicit inputs and outputs remain unknown.

## Acceptance examples

These are conditional static consequences, not executed tests. They assume the specified accesses and calls are reached and complete.

1. **Distinct fixed-location references.**  
   Suppose the captured seed is 16, the store-address reference yields 32, and the store-data reference yields 48. The prefix stores the 32-bit value 48 at address 64, then reads a byte at address 32. This condition does not prescribe which of the two store-expression references is evaluated first.

2. **Stable ordinary fixed location.**  
   If all three fixed-location references yield 16, the word store is at address 32 and the subsequent byte read is also at address 32. The byte value still depends on endianness and the memory’s access semantics.

3. **Zero branch value.**  
   The prefix stores zero at the initial derived base plus 100. The auxiliary-base writes are skipped, and F-U10 uses the entry link plus 0x6BE. The later invalid-instruction marker does not imply successful completion.

4. **Carry-set transfer and selector addressing.**  
   With a nonzero branch value, set carry, and source byte 166 at the branch value plus 5, that byte is stored at the auxiliary base plus 5 before F-fa0c227e09d7. If the captured selector is 7, the subsequent selector-based word read is at address 91, not at the fourth argument plus 91.

5. **Unsigned initial base.**  
   If the initial halfword is 65,535, the derived base is 65,535 rather than a sign-extended negative value. The prefix’s branch-value word store is at address 65,635.

6. **Replacement-base store and result order.**  
   If the replacement halfword is 4,096 and the selector is 3, the carry-clear path stores byte 3 at address 4,099 after F-U06. If F-U06 exposes low portion 16 and high portion 32, the following halfword read is at address 48. F-U07 receives 16, 32, the represented negation of 16, and that halfword.

7. **Repeated targets remain separate invocations.**  
   On the carry-clear path, calls to F-U03 and F-U04 both target 0x08573CA3, with the offset-20 store between them. On the carry-set path, F-U13 and F-U14 both target 0x08573BD3, with the offset-116 store between them.

8. **Fixed carry-set final target.**  
   If the carry-set path reaches its final call, F-U10’s represented target is 0x085733C7 regardless of the entry link value. This does not apply to the zero-value path.

9. **Aliased initial byte destinations.**  
   If the third argument plus the captured seed equals the third argument plus 5, the entry-byte store occurs first and the captured-byte store second. Under ordinary writable-memory semantics without an intervening external change, the second determines the byte immediately afterward.

## Uncertainties

The following remain documented_only:

- Application purpose, intended structures, and valid input domains.
- Required initialization of implicit entry state.
- Memory mappings, alignment, endianness, and ordinary versus special access semantics.
- Stability of the fixed-location value and the number of physical loads.
- Relative evaluation order of the two references within the initial word-store expression.
- Exact overflow, address wrapping, and negation behavior.
- Dependency behavior, ABI details, implicit arguments, and return-side state interpretation.
- Executability and instruction-set interpretation of calculated targets.
- Entry-link-dependent target values and F-U11’s unrecovered target.
- Dependency ownership, retention, release, global mutation, and failure behavior.
- Fault handling for invalid accesses or call targets.
- Whether the terminal invalid instructions indicate intentional traps, invalid decoding, or missing control flow.
- Whether the represented terminal transfer can return normally.

The fixed target calculations and target equalities are not unresolved merely because callee behavior is unknown. Conversely, known target arithmetic does not establish what a dependency does.

## Evidence

This contract is based solely on the supplied static representation. It supports the stated widths, represented value references, access ordering, branch predicates, argument construction, and call-target arithmetic.

The representation contains an invalid-instruction warning and an unrecovered indirect transfer. Those limitations prevent a complete claim about executable control flow or normal termination.

No runtime execution, dependency inspection, memory-map validation, ABI verification, or independent instruction analysis has been performed. The acceptance examples express conditional implications of static evidence, not observed results.
