<!-- Trusted provenance: F-4705edbd1f6b; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-4705edbd1f6b

## Purpose

The supplied static evidence establishes an operation dominated by an ordered chain of calls and coprocessor transfers. Its final explicit data-memory effect is a 16-bit write to an address derived from a dependency’s 64-bit result. The visible path then reaches an invalid-instruction termination marker rather than an ordinary return.

No application-level purpose is established. In particular, the evidence does not justify describing this operation as initialization, decoding, allocation, device configuration, or data processing.

The source representation explicitly warns that instruction decoding and control-flow recovery encountered invalid data. Consequently, this contract distinguishes:

- **Established static evidence:** operations, dependencies, and ordering explicitly represented in the supplied material.
- **Conditional interpretation:** consequences of interpreting the displayed integer operations as fixed-width machine operations.
- **documented_only unknowns:** behavior that cannot be established without instruction-level, dependency, platform, or runtime evidence.

No runtime tests have been performed.

## Interface

### Explicit inputs

The displayed interface has four 32-bit argument positions:

| Position | Representation | Visible use |
|---|---|---|
| First | Uninterpreted 32-bit value | No explicit use in the recovered body. |
| Second | Uninterpreted 32-bit value | No explicit use in the recovered body. |
| Third | Signed 32-bit integer | Supplies several left-shifted call arguments. |
| Fourth | Signed 32-bit integer | Supplies a value shifted left by 31 bits. |

The absence of explicit uses for the first two arguments does not prove that their incoming register contents are unobservable to indirectly called code.

No input is established as a pointer, buffer, length, handle, or owned object.

### Additional execution-state inputs

The four arguments are insufficient to describe the visible operation. It also depends on:

- An incoming link-related value used to form the first indirect call target.
- Two incoming general-register values whose definitions are absent.
- Two additional register values exposed by earlier calls, without a complete return-value contract.
- Unresolved operands associated with coprocessor state.
- The behavior and return values of numerous unresolved call targets.

These are environmental preconditions, not additional documented application arguments.

### Output and completion

The displayed interface declares no return value. The visible path contains no ordinary return.

If execution reaches the end of the recovered path, it attempts the final halfword store and then reaches the invalid-instruction termination marker. The concrete platform exception, process outcome, and possibility of reaching that point are **documented_only**.

### Arithmetic notation

For the conditional fixed-width interpretation:

- \(L_k(x)\) means the low 32 bits of \(x\) shifted left by \(k\).
- \(\operatorname{low}(w)\) and \(\operatorname{high}(w)\) denote the lower and upper 32-bit halves of a 64-bit value.
- Arithmetic right shift interprets its operand as signed 32-bit before shifting.
- Unsigned right shift inserts zero bits.

These definitions express bit-level consequences; they do not establish that overflowing signed shifts are valid in a source-language execution model.

## Behavior

### Dependency identification and sequencing

The one dependency with a supplied identity is **F-3de949054bdd**.

The other calls are unresolved. This contract assigns them opaque documentary IDs **F-U01** through **F-U66**, in their visible order, excluding F-3de949054bdd. These IDs identify call occurrences, not verified distinct functions. Two occurrences might ultimately resolve to the same implementation.

The sequence below is the visible-path order. Each later event is conditional on earlier calls and transfers allowing execution to continue. “No explicit arguments” means that the recovered call expression supplies none; it does not establish that the callee observes no machine state.

### 1. Initial calls and argument-derived values

F-U01 executes first, using the target derived from incoming link-related state. F-U02 follows. Neither has explicit arguments.

F-U03 receives two 32-bit values derived from an unresolved incoming register word:

- The first is the result of masking that word to 26 bits, shifting right by 9, then shifting left by 26.
- The second is that word shifted left by 22.

Under fixed-width semantics, the first value retains incoming bits 9–14 in output bits 26–31. The second retains incoming bits 0–9 in output bits 22–31.

F-U04 receives F-U03’s 32-bit result.

F-U05 then receives, in order:

1. The fourth explicit input shifted left by 31.
2. An unresolved extra register result associated with preceding execution.
3. The third explicit input shifted left by 26.
4. The third explicit input shifted left by 2.

A coprocessor transfer uses F-U05’s result before F-U06 receives that same result.

F-U07 produces a value passed to F-U08. F-U08’s result is passed to F-U09. F-U10, F-U11, F-U12, and F-U13 then execute in that order without explicit arguments. F-U14 receives F-U13’s result.

F-U15 receives an explicit zero. F-U16 and F-U17 follow without explicit arguments.

### 2. A signed result and the identified dependency

F-U18 receives three values:

1. A second unresolved incoming register word shifted left by 31.
2. Another unresolved extra register result.
3. Zero.

F-U19 and F-U20 follow without explicit arguments. F-U21 then returns a value represented as signed 32-bit.

F-U22 receives:

- F-U21’s result.
- The third explicit input shifted left by 26 and then left by 22.
- F-U21’s result shifted left by 5.

The second argument is zero under 32-bit truncating shift semantics. This is not an independently validated property of the original instructions.

F-U23 executes without explicit arguments, and F-U24 receives its result. **F-3de949054bdd** executes immediately afterward, with no explicit arguments and no visible use of a return value.

F-U25 then produces a value consumed by F-U26. F-U27 produces a value consumed by F-U28.

F-U29 receives two arguments:

- The third explicit input shifted left by 26 and then left by 19.
- An arithmetic right shift by 5 of the previously computed 32-bit value obtained by shifting F-U21’s result left by 5.

The first is zero under fixed-width semantics. The second is not generally equal to F-U21’s original result: it preserves the low 27 bits and sign-extends bit 26.

F-U30 receives F-U29’s result.

### 3. First 64-bit result and two consecutive transfers

F-U31 produces a 64-bit value. Its low half is supplied to a coprocessor transfer. F-U32 then receives its high half shifted left by 1.

F-U33 executes without explicit arguments. Its result is used by two consecutive coprocessor transfers with different unresolved state operands.

F-U34 executes next. Its result, shifted left by 27, supplies the third argument of F-U35; the other three arguments are zero.

Only the low five bits of F-U34’s result contribute to that shifted value under fixed-width semantics.

### 4. Zero-producing argument chains

F-U36 through F-U41 execute in order, without explicit arguments. A coprocessor transfer consumes F-U41’s result.

F-U42, F-U43, and F-U44 then execute without explicit arguments.

The value formed from F-U34’s result by shifting left by 27 is shifted left by another 24. This value is used first by a coprocessor transfer and then as F-U45’s first argument. F-U45’s second argument is zero. Under fixed-width semantics, both arguments are zero.

F-U46 and F-U47 execute without explicit arguments. A coprocessor transfer consumes F-U47’s result. F-U48 follows.

F-U49 receives three arguments whose displayed definitions all reduce to zero. Its signed 32-bit result determines several later call arguments and transfer values.

A coprocessor transfer uses F-U49’s result shifted left by 1.

F-U50 then receives:

1. F-U49’s result shifted left by 1.
2. The value derived from F-U34 by shifting left by 27, masking to 23 bits, then shifting left by 4.
3. F-U49’s result shifted left by 4.
4. Zero.

The second argument is zero under fixed-width semantics because the mask removes every bit that could survive the earlier left shift.

### 5. Further transfers and a second 64-bit result

F-U51 and F-U52 execute without explicit arguments. Two coprocessor transfers follow, both carrying zero.

F-U53 receives zero and returns a 32-bit value. A subsequent coprocessor transfer carries F-U49’s result shifted left by 4 and then by 8.

F-U54 receives:

- F-U49’s result shifted left by 4 and then by 8.
- F-U49’s result shifted left by 4 and then by 12.
- F-U53’s result shifted left by 9.

F-U55 produces a 64-bit value. F-U56 receives its low half, its high half, and its low half shifted left by 1, in that order.

An additional intermediate quantity is formed by masking the value “F-U49’s result shifted left by 4” with 3, then shifting left by 16. Its masked input is necessarily zero under fixed-width semantics.

F-U57 receives that quantity shifted left by 8, and therefore receives zero under that interpretation. F-U58 consumes F-U57’s result; F-U59 consumes F-U58’s result.

### 6. Final dependency chain and address derivation

F-U60 executes without explicit arguments.

F-U61 produces a 64-bit value. F-U62 receives its low half, its high half, and its high half shifted left by 11.

F-U62 also produces a 64-bit value. This result supplies both:

- The low 32-bit value passed to F-U63.
- The bits used to derive the eventual store address.

F-U64 executes without explicit arguments. F-U65 receives F-U64’s result and the previously zero-producing intermediate quantity shifted left by a further 25 bits. Its second argument is zero under fixed-width semantics.

F-U66 executes without explicit arguments.

The final store occurs after F-U66. No explicit subsequent read of a dependency result changes the retained address calculation in the recovered representation. This does not assert that any dependency preserves global state or memory.

### 7. Final halfword store and termination

Let \(W\) be F-U62’s 64-bit result, interpreted as an unsigned bit pattern. The displayed address calculation first shifts \(W\) right by 41 and converts the result to 32 bits. That result fits in 23 bits, so its later signed interpretation is nonnegative.

The complete address reduces to:

\[
q = W \mathbin{\text{unsigned-right-shift}} 52,
\qquad
A = q + \left\lfloor q/4 \right\rfloor.
\]

Thus:

- Only bits 52–63 of F-U62’s result determine the address.
- \(q\) ranges from 0 through 4095.
- \(A\) ranges from 0 through 5118, inclusive.
- No base pointer is added.
- The address need not be aligned to two bytes.

The attempted write stores the 16-bit value **0x00CD** at address \(A\). It is not a one-byte write. Byte order is not established.

After this store, the recovered path reaches the invalid-instruction termination marker. There is no visible success result, cleanup path, or ordinary return.

## Side effects

### Calls

All dependencies may have side effects not visible here. Their names, arguments, or unused return values do not establish purity, memory ownership, allocation behavior, global-state preservation, or failure behavior.

The explicit call ordering is observable if those dependencies interact with shared state or devices.

### Coprocessor transfers

Eleven coprocessor transfers appear in this order:

| Position | Data supplied | Displayed numeric operation fields | Unresolved state-operand associations |
|---|---|---|---|
| After F-U05 | F-U05 result | 0, 6, 7 | 3 and 15 |
| After F-U31 | Low half of F-U31 result | 0, 6, 7 | 3 and 15 |
| After F-U33, first | F-U33 result | 0, 3, 7 | 6 and 15 |
| After F-U33, second | Same result | 0, 3, 7 | 15 and 15 |
| After F-U41 | F-U41 result | 0, 1, 7 | 1 and 15 |
| After F-U44 | Zero under fixed-width interpretation | 0, 5, 7 | 5 and 15 |
| After F-U47 | F-U47 result | 0, 6, 7 | 9 and 15 |
| After F-U49 | F-U49 result shifted left by 1 | 0, 5, 7 | 10 and 15 |
| After F-U52, first | Zero | 0, 7, 7 | 9 and 15 |
| After F-U52, second | Zero | 0, 0, 7 | 15 and 15 |
| After F-U53 | F-U49 result shifted left by 12 overall | 0, 2, 7 | 9 and 15 |

The table records the supplied representation, not a validated architectural instruction encoding. The affected hardware facilities, privilege requirements, barriers, and trap behavior are **documented_only**.

### Memory and ownership

The final halfword store is the only explicit ordinary data-memory access in the recovered body. Calls may perform arbitrary additional accesses.

No allocation, release, ownership transfer, or buffer layout is established. In particular, a value passed from one dependency to another must not be assumed to be an owned pointer.

The final destination is a computed numeric address, not a validated caller-provided buffer. Its accessibility, writability, and alignment are unchecked in the visible path.

## Dependencies

**F-3de949054bdd** is invoked once between F-U24 and F-U25. Its behavior, return convention, side effects, and completion guarantees are unknown.

**F-U01–F-U66** are unresolved indirect call occurrences. The evidence establishes their visible order and explicit argument relationships, but not their complete interfaces.

The first unresolved target depends on incoming execution state. Subsequent targets are expressed through code-location-derived arithmetic in the supplied representation. That is insufficient to establish valid entry points, executable mappings, or a dependency inventory.

No dependency is assumed to:

- Return normally.
- Preserve global or device state.
- Validate arguments.
- Preserve or release memory.
- Interpret a 32-bit value as signed, unsigned, or pointer-valued.
- Follow the displayed return-width convention correctly.

Any such property requires additional evidence.

## Acceptance examples

These are conditional acceptance examples for a compatible implementation or execution model, not observed test results.

1. **Final address zero.**  
   If F-U62 returns a value whose upper 12 bits are zero, and all intervening operations complete, the final write targets address 0 and stores a 16-bit 0x00CD. A null-address fault is not prevented by the visible path.

2. **Address gap.**  
   If those upper 12 bits are 3, the destination is 3. If they are 4, the destination is 5. Address 4 is skipped by this mapping. No alignment adjustment occurs.

3. **Largest destination.**  
   If F-U62’s upper 12 bits are all set, the destination is 5118. Altering its lower 52 bits does not change that destination.

4. **Signed reconstruction is not identity.**  
   If F-U21 returns the 32-bit pattern 0x04000000, shifting left by 5 gives 0x80000000. Arithmetic right shift by 5 yields 0xFC000000. That is the second argument to F-U29 under the stated machine interpretation, not the original positive value.

5. **Narrow incoming-bit dependence.**  
   Changing the unresolved register word used by F-U03 only outside bits 0–14 leaves both explicit arguments to F-U03 unchanged. Other observable effects of changing incoming machine state remain unknown.

6. **64-bit result splitting.**  
   If F-U55 returns 0x1122334455667788, F-U56 receives 0x55667788, 0x11223344, and 0xAACCEF10 in that order under fixed-width semantics.

7. **Failure before the store.**  
   If a dependency does not return or a coprocessor transfer traps, the contract does not require later calls, the final store, or the termination marker to occur.

## Uncertainties

The following remain **documented_only**:

- Whether the recovered path corresponds to reachable executable instructions.
- Whether the displayed call boundaries and argument lists are accurate.
- Whether signed shifts should be interpreted as truncating machine operations.
- The calling convention, instruction-set mode, privilege level, and endianness.
- The definitions of incoming unresolved registers and extra register results.
- The meaning of coprocessor operations and unresolved state operands.
- The identities and semantics of unresolved dependencies.
- Whether the final halfword store can complete at low or unaligned addresses.
- The concrete behavior represented by the invalid-instruction termination marker.
- Any application-level purpose, error reporting convention, or recovery protocol.

There are no visible conditional branches, validation checks, retries, or explicit error returns in the recovered path. That observation does not establish their absence from dependencies or unrecovered execution.

## Evidence

This contract uses only the supplied static representation and the provided dependency mapping.

Established observations include the four displayed 32-bit arguments, the ordered calls, the single identified dependency, eleven coprocessor transfers, the final 16-bit store, and the invalid-instruction termination marker.

Arithmetic reductions and acceptance examples are conditional deductions from the displayed 32-bit and 64-bit operations. They are not runtime observations.

No implementation access, instruction-level validation, dependency inspection, or runtime testing has been performed.
