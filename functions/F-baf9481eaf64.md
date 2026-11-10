<!-- Trusted provenance: F-baf9481eaf64; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-baf9481eaf64

## Purpose

The established behavior is a conditional sequence of external invocations and coprocessor transfers, ending in a 16-bit memory write to a numerically constructed address. The supplied representation ends at an invalid-instruction boundary rather than an ordinary return.

No application-level purpose is established. Initialization, decoding, allocation, and device configuration would be interpretations, not supported functional descriptions.

This contract describes the represented path, not a verified executable trace. A control-flow warning applies to the supplied material. Every later effect is conditional on preceding operations completing and control continuing along that path.

## Interface

### Explicit inputs

| Input | Represented width | Established use |
|---|---:|---|
| First input | 32 bits, signed interpretation | Contributes a shifted bit pattern to one external invocation. |
| Second input | 32 bits, unspecified interpretation | No explicit consumption appears in the body. |
| Third input | 32 bits, unsigned interpretation | Passed unchanged initially and subsequently used in shifted and masked forms. |

An unused explicit input is not necessarily irrelevant to machine execution. An external operation could consume incoming register state not described by its apparent argument list. The actual calling convention is unresolved.

### Additional incoming context

The represented behavior also requires:

- An incoming 32-bit register value used in the first invocation.
- An incoming link/control value used to determine that invocation’s target.
- Five incoming coprocessor-context values. This contract calls them context operands A through E solely to distinguish their reuse.

These are additional machine-context inputs, not values derived from the three explicit inputs.

### Results and completion

The represented interface has no ordinary return value. No normal return is shown.

The last ordinary memory effect is a halfword write containing decimal 205, or hexadecimal 00CD. An invalid-instruction boundary follows. The evidence does not establish whether actual execution traps there, follows a different decoding, or behaves otherwise.

### Width and interpretation rules

Ordinary argument calculations and retained ordinary results are interpreted as 32-bit bit patterns. Three consumed external results are represented as 64-bit quantities.

For the arithmetic described below:

- Left shifts retain only the low 32 bits unless explicitly described as 64-bit extraction.
- A “low word” is bits 0–31 of a 64-bit quantity.
- A “high word” is bits 32–63.
- Unsigned right shifts introduce zeros.
- Word splitting is numerical; it does not specify byte order or memory layout.

These rules express a machine-width interpretation of the representation. They do not establish that equivalent signed operations would be defined in a high-level language, or that the original instruction semantics have been recovered correctly.

## Behavior

### Observable sequence

There are 32 represented external invocations. This contract assigns them opaque IDs F-000000000001 through F-000000000032 in invocation order. These identify call occurrences, not independently verified function identities. Different occurrences are not proven to have distinct targets.

The external invocations occur in increasing ID order, interleaved with the seven coprocessor transfers specified under **Side effects**. The final store follows F-000000000032.

No conditional branch, retry, status-based alternative, or cleanup path appears in the represented body. This does not establish that dependencies return normally or that processor operations are legal.

An empty explicit argument list means only that no arguments are displayed. It does not establish absence of implicit inputs, memory effects, or relevant results outside the represented body.

### Initial and explicit-input-dependent values

F-000000000001 receives four arguments:

| Position | Argument |
|---:|---|
| 1 | The incoming implicit register bit pattern shifted left by 21 positions. |
| 2 | Zero. |
| 3 | The unchanged third explicit input. |
| 4 | Zero. |

Only the low 11 bits of the implicit register contribute to the first argument. The invocation target is constructed from the incoming link/control value plus an offset of 3494. The target-address representation, validity, and any required instruction-state interpretation are unresolved.

F-000000000011 receives two arguments: the third input shifted left by 24 positions, followed by the first input shifted left by 27 positions. Consequently, only the low eight bits of the third input and the low five bits of the first input affect this invocation’s displayed arguments.

The third input also contributes to F-000000000016, but through a different transformation: its low 23 bits are retained and then shifted left by four positions. The resulting argument cannot exceed hexadecimal 07FFFFF0.

### Retained ordinary result and zero invariants

F-000000000015 receives three zero arguments. These zeros follow from a represented local value initialized to zero and not reassigned before that invocation.

Call its consumed 32-bit result the **working result**. Two forms of this result recur:

- The **doubled pattern** is the working result shifted left by one position.
- The **scaled pattern** is the working result shifted left by four positions.

Both are truncated to 32 bits. These are bit-pattern transformations, not guarantees of signed mathematical multiplication without overflow.

F-000000000016 receives, in order:

1. The doubled pattern.
2. The masked-and-shifted third-input value described above.
3. The scaled pattern.
4. Zero.

The low four bits of the scaled pattern are necessarily zero. A later intermediate uses only its low two bits, shifted left by 16 positions; this intermediate is therefore zero for every possible working result.

That invariant determines two later arguments:

- F-000000000023 receives zero.
- F-000000000031 receives zero as its second argument, even though an additional left shift intervenes in the represented calculation.

This is a consequence of the displayed value flow. It is not a general claim that unknown dependencies preserve processor state.

F-000000000019 receives one zero argument and supplies another consumed 32-bit result. F-000000000020 receives three shifted patterns: the scaled pattern shifted left by eight positions, the scaled pattern shifted left by 12 positions, and the result of F-000000000019 shifted left by nine positions.

### Result-processing relationships

The remaining consumed-result relationships are:

| Producer | Consumer and argument contract |
|---|---|
| F-000000000007 | Its 32-bit result supplies the first coprocessor transfer. |
| F-000000000013 | Its 32-bit result supplies the third coprocessor transfer. |
| F-000000000021 | Its represented 64-bit result supplies F-000000000022: low word, high word, then low word shifted left by one. |
| F-000000000023 | Its 32-bit result is the sole displayed argument to F-000000000024. |
| F-000000000024 | Its 32-bit result is the sole displayed argument to F-000000000025. |
| F-000000000027 | Its represented 64-bit result supplies F-000000000028: low word, high word, then high word shifted left by 11. |
| F-000000000028 | Its represented 64-bit result supplies both the argument to F-000000000029 and the final-address calculation. |
| F-000000000030 | Its 32-bit result is the first displayed argument to F-000000000031. |

F-000000000029 receives only the low word of the result of F-000000000028. F-000000000031 receives two arguments: the result of F-000000000030 and zero.

No displayed result from F-000000000029, F-000000000031, or F-000000000032 contributes to the final address. Their other effects remain unknown.

### Final address and write

The address calculation depends only on the represented 64-bit result of F-000000000028.

An unsigned right shift first extracts its upper 23 bits. That quantity fits within a positive signed 32-bit value. A subsequent arithmetic right shift by 11 therefore has the same result as a logical right shift.

Let \(k\) denote the top 12 bits of that 64-bit result. Then:

\[
0 \leq k \leq 4095,
\qquad
\text{address}=k+\left\lfloor\frac{k}{4}\right\rfloor.
\]

The calculation has no explicit allocation-derived base, relocation, or buffer pointer. Its possible start addresses lie between 0 and 5118 inclusive. Not every integer in that interval is attainable: for example, \(k=3\) gives address 3, while \(k=4\) gives address 5.

Neither addition nor the final shifts overflow under this interpretation. The lower 52 bits of the wide result do not affect the address.

After F-000000000032 returns along the represented path, the body writes the 16-bit value 205 at this address. No preceding check establishes that the address is nonzero, aligned, mapped, writable, or within an owned object.

## Side effects

### Coprocessor transfers

Seven transfers are represented. The selector tuples below reproduce the numeric selectors without assigning hardware meanings to their positions.

Context operands A–E denote the five incoming coprocessor-related values. Repeated letters indicate reuse of the same represented incoming value.

| Transfer | Ordering position | Selector tuple | Data operand | Context operands |
|---:|---|---|---|---|
| 1 | After F-000000000007, before F-000000000008 | (0, 1, 7) | Result of F-000000000007 | A, E |
| 2 | After F-000000000010, before F-000000000011 | (0, 5, 7) | Third input shifted left by 24 | B, E |
| 3 | After F-000000000013, before F-000000000014 | (0, 6, 7) | Result of F-000000000013 | C, E |
| 4 | After F-000000000015, before F-000000000016 | (0, 5, 7) | Doubled pattern | D, E |
| 5 | After F-000000000018, before F-000000000019 | (0, 7, 7) | Zero | C, E |
| 6 | After transfer 5, before F-000000000019 | (0, 0, 7) | Zero | E, E |
| 7 | After F-000000000019, before F-000000000020 | (0, 2, 7) | Scaled pattern shifted left by eight | C, E |

The zero-valued intermediate is calculated between transfers 5 and 6, but it is not the data operand of either transfer: both explicitly carry zero.

Instruction legality, privilege requirements, architectural meaning, and resulting hardware state are unresolved. The represented sequence does not establish memory barriers, external visibility timing, or atomicity.

### Ordinary memory

The final halfword write is the only explicit ordinary data-memory access in the represented body. No ordinary data-memory load is displayed outside dependency execution. Instruction fetches, implicit processor accesses, and dependency memory activity are not covered by that statement.

The written numeric value is 00CD:

- On a little-endian target, the lower-addressed byte would be CD and the next byte 00.
- On a big-endian target, those byte values would be reversed.

Neither byte order is established. Under an ordinary byte-addressed two-byte layout, the greatest addressed byte would be 5119. That layout does not prove either byte is accessible.

Odd start addresses are possible. The representation supplies no alignment handling. Success, faulting, or another target-specific result remains unresolved.

### Ownership and state

No allocation, release, ownership transfer, or buffer extent is established. The numeric destination is not proven to identify valid storage.

Dependencies may read or modify globals, memory, processor state, device state, and ownership state. Their argument lists provide no basis for assuming preservation or release behavior. No rollback or cleanup is displayed if an earlier operation prevents the final store.

## Dependencies

All 32 dependency occurrences are unresolved.

The invocations with no displayed arguments are:

- F-000000000002 through F-000000000010.
- F-000000000012 through F-000000000014.
- F-000000000017 and F-000000000018.
- F-000000000021.
- F-000000000026 and F-000000000027.
- F-000000000030 and F-000000000032.

All other displayed argument contracts are specified under **Behavior**.

Consumed 32-bit results come from F-000000000007, F-000000000013, F-000000000015, F-000000000019, F-000000000023, F-000000000024, and F-000000000030. Consumed represented 64-bit results come from F-000000000021, F-000000000027, and F-000000000028.

These widths describe caller-side use, not verified external API declarations. Actual signatures, calling conventions, hidden inputs, acceptable argument domains, and return conventions are unknown.

The first target depends on incoming context. The remaining targets are represented through fixed address constructions, but their identities and semantics are not established here.

No explicit error-status examination is shown. A dependency that traps, fails to return, or redirects control prevents any guarantee that subsequent events occur. A returned value is used according to the described bit-pattern rules without a displayed success/failure classification.

## Acceptance examples

These are conditional static acceptance examples, not executed tests. Each assumes the represented widths and sufficient continuation to reach the event being described.

1. **Explicit-input truncation.**  
   With first-input bit pattern 0000003F and third-input bit pattern 000001AB, F-000000000011 receives AB000000 followed by F8000000. The discarded higher bits do not contribute.

2. **Different treatment of the third input.**  
   With third input FFFFFFFF, the second coprocessor transfer carries FF000000, while the second argument to F-000000000016 is 07FFFFF0. The latter is not FFFFFFF0 because only 23 input bits are retained.

3. **Working-result wraparound.**  
   If F-000000000015 produces 80000001, the doubled pattern is 00000002 and the scaled pattern is 00000010. Transfer 4 carries 00000002, and F-000000000016 receives 00000010 as its third argument.

4. **Unconditional zero consequence within the represented arithmetic.**  
   Any 32-bit working result produces a scaled pattern whose low two bits are zero. F-000000000023 therefore receives zero, and F-000000000031 receives zero as its second argument.

5. **Wide-result word ordering.**  
   If F-000000000021 produces 0123456789ABCDEF, F-000000000022 receives 89ABCDEF, 01234567, and 13579BDE, in that order. This example specifies numerical word extraction, not byte layout.

6. **Null and unaligned destinations remain possible.**  
   A top-12-bit value of zero selects address zero. A value of one selects address one. Neither case has a displayed rejection path; actual store success depends on the target.

7. **Address boundaries and gaps.**  
   Top-12-bit values of 3, 4, and 4095 select addresses 3, 5, and 5118. Changing only the lower 52 bits leaves the destination unchanged.

8. **No premature write.**  
   If F-000000000032 does not return, the final store is not reached on the represented path. There is no displayed alternate write or cleanup.

9. **Completion is not an ordinary return.**  
   Even if the halfword write succeeds, the next represented boundary is invalid instruction data. A caller-visible normal return is not an acceptance requirement supported by this evidence.

## Uncertainties

The following remain **documented_only**:

- Validity of the represented control flow and instruction decoding.
- Actual behavior at and after the invalid-instruction boundary.
- Target architecture, address representation, byte order, memory map, alignment behavior, and privilege state.
- Complete incoming machine state and calling conventions.
- Dependency identities, aliasing between call targets, side effects, and failure behavior.
- Whether represented wide results correspond to actual 64-bit return values.
- Whether the coprocessor operations are executable in the intended environment.
- Whether the low numeric destination denotes ordinary storage, device memory, inaccessible space, or a decoding artifact.
- Whether any caller-visible completion or error mechanism exists outside this body.

An orchestration role involving processor state is plausible **inferred intent**, because external calls and coprocessor transfers dominate the representation. No more specific purpose is established.

## Evidence

**Established static evidence:** the supplied representation contains three explicit parameters, additional incoming machine-context values, 32 ordered external invocations, seven coprocessor transfers, one final 16-bit store of 205, and an invalid-instruction boundary without an ordinary return.

**Derived static consequences:** the scaled pattern forces the later intermediate to zero; the final destination depends only on the top 12 bits of one represented 64-bit result; destination start addresses have bounds 0 and 5118, with gaps inside that interval.

**Evidence limits:** this contract uses only the supplied representation. No independent implementation inspection, instruction-level validation, dependency inspection, runtime execution, or tests have been performed.
