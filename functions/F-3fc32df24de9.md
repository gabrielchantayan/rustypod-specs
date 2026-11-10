<!-- Trusted provenance: F-3fc32df24de9; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-3fc32df24de9

## Purpose

F-3fc32df24de9 performs an ordered sequence of indirect calls, computed-address word accesses, and two byte writes. The supplied static representation shows one input, no explicit result, and an abnormal terminal boundary after the final indirect call.

The established behavior is low-level memory and call sequencing. No application-level purpose, object model, allocation protocol, or successful completion condition is established. In particular, the repeated calls must not be interpreted as allocation, initialization, serialization, or synchronization without additional evidence.

This contract distinguishes:

- **Established static evidence:** operations and ordering explicitly represented in the supplied material.
- **Inferred interpretation:** a 32-bit bit-vector model used to describe those operations consistently.
- **documented_only unknowns:** unresolved calling conventions, dependency behavior, memory validity, and the actual terminal instruction behavior.

All later operations are conditional on earlier memory accesses succeeding and earlier dependencies returning normally.

## Interface

### Input

The visible interface accepts one integer input, designated **P** in this document. Its represented width is 32 bits. Both its bit pattern and its use in address arithmetic matter:

- It is written as a complete word.
- It participates in multiplication by four to obtain an address.
- It is explicitly passed to the first indirect dependency.

There is no visible validation of P, no null-like sentinel handling, and no range check.

The routine also depends on implicit entry context: the first indirect call target is calculated from an incoming return-link value plus a fixed displacement. The return-link value is therefore an execution prerequisite even though it is not a declared argument.

### Output and termination

The visible interface declares no return value. No normal return is represented. After the final indirect call returns, the supplied representation ends at an invalid-instruction boundary.

This does **not** establish a particular hardware exception, operating-system signal, diagnostic, exit status, or nonreturning runtime service. It establishes that successful return to the caller cannot be specified from the available material.

### Width and arithmetic model

The following notation expresses the represented operations:

- **W(x)** means the low 32 bits of x.
- **A(x)** means the 32-bit address W(4x + 20).
- **B(x)** means the 32-bit address W(16x + 20).
- **C(x)** means the 32-bit address W(268435456x + 1).

Word accesses are represented as 32-bit accesses. Byte reads produce unsigned values from 0 through 255. Byte writes retain the low eight bits of their source value.

Address arithmetic and left shifts below use a 32-bit bit-vector interpretation, retaining low-order bits. This is an explicit interpretation of the low-level representation, not a claim that signed overflow or left-shifting a negative signed integer has portable high-level-language semantics. The actual instruction widths and overflow behavior remain subject to architecture confirmation.

The one right shift of a loaded word is unsigned. The later masked right shift is also applied to an unsigned value.

### Preconditions

Reaching the final boundary requires:

- Every executed call target to be valid and callable under the actual ABI.
- Every dependency reached before that boundary to return normally.
- Every computed memory access to have the required read or write permission and any required alignment.
- Entry context sufficient to resolve the first indirect call.
- Any hidden dependency requirements to be satisfied.

No buffer extent, object ownership, or accessible address range is supplied by the visible interface.

## Behavior

### Dependency naming and captured results

The dependency IDs **F-U01** through **F-U44** below identify indirect call sites in execution order. They are document-local opaque IDs, not resolved function identities. Different IDs do not prove different runtime targets.

The following symbols denote values captured from particular calls, rather than storage locations:

| Symbol | Source | Represented use |
|---|---|---|
| R1 | F-U05 result | Index for a word write and a scaled call argument |
| R2 | F-U10 result | Index for a word write, a word read, and a scaled argument |
| R3 | F-U15 result | Index for a word write, bit-field arguments, and both byte-read addresses |
| R4 | F-U20 result | Index for a word write and the value later copied by another word write |
| R5 | F-U25 result | Index for a word write and a scaled argument |
| R6 | F-U30 result | Unsigned source for both byte-write values and three call arguments |
| R7 | F-U32 result | Opaque 32-bit value passed to F-U33 |
| R8 | F-U37 result | Source for a shifted argument |
| R9 | F-U42 result | Source for a shifted argument |

Results from other calls are not explicitly consumed in the visible representation. This does not establish that those calls lack side effects.

### 1. Initial write and first result

Before any dependency invocation, the routine writes the complete 32-bit P value to A(P).

It then invokes, in order:

1. F-U01 with explicit argument P.
2. F-U02, F-U03, and F-U04 with no explicit arguments shown.
3. F-U05 with no explicit arguments shown, capturing R1.

The initial word write is observable before F-U01 is entered. Any effects of these dependencies occur before the next direct memory write.

### 2. Second word write and sampled word

The routine writes P to A(R1), then invokes:

1. F-U06 with W(4R1).
2. F-U07, F-U08, and F-U09 with no explicit arguments shown.
3. F-U10 with no explicit arguments shown, capturing R2.

It then writes W(4R2) to B(R2).

**After that write**, it reads a 32-bit unsigned word from A(R2). Denote the captured value **V**. The write and read use different address formulas; they must not be merged or reordered merely because both derive from R2.

Their addresses coincide exactly when W(12R2) is zero. Under the 32-bit model, this occurs when R2 is a multiple of 2³⁰. Otherwise, they refer to distinct starting addresses.

V is a snapshot taken at this point. Later modifications of its source address do not change the captured value.

### 3. Shifted-word write

After capturing V, the routine invokes:

1. F-U11 with W(4R2).
2. F-U12, F-U13, and F-U14 with no explicit arguments shown.
3. F-U15 with no explicit arguments shown, capturing R3.

It writes the unsigned value V shifted right by 12 bits to A(R3). The stored value ranges from zero through 1,048,575.

It then invokes:

1. F-U16 with W(4R3).
2. F-U17, F-U18, and F-U19 with no explicit arguments shown.
3. F-U20 with no explicit arguments shown, capturing R4.

The routine next writes W(4R4) to B(R4).

### 4. Three-argument call and copied scaled value

After the preceding word write, F-U21 is invoked with these three explicit arguments, in order:

1. W((W(4R3) AND 63) shifted left by 14).
2. W(4R4).
3. W(R3 shifted left by 30).

The first argument selects bits from the scaled result W(4R3), not directly from V. Because W(4R3) has its bottom two bits clear, this argument has only 16 possible values: multiples of 65,536 from zero through 983,040.

The third argument retains only the bottom two bits of R3, positioned in the top two bits of a 32-bit word.

The routine then invokes F-U22, F-U23, and F-U24 with no explicit arguments shown, followed by F-U25, whose result is R5.

It writes the previously computed W(4R4) to A(R5). This is not a fresh read from B(R4). Dependency changes to that memory location do not alter the captured scaled value.

### 5. First byte-directed write

The routine invokes:

1. F-U26 with W(4R5).
2. F-U27, F-U28, and F-U29 with no explicit arguments shown.
3. F-U30 with no explicit arguments shown, capturing unsigned R6.

It then reads one unsigned byte from C(R3). Denote this first byte **Q1**.

The routine writes the low eight bits of W(R6 shifted left by two) to address Q1 + 17.

Consequences of the byte-sized indirection are established:

- The destination lies between address 17 and address 272, inclusive.
- Q1 is used as an unsigned numeric address component, not as a full-width pointer.
- The written byte is a multiple of four and ranges from zero through 252.
- There is no represented check that this low-address write is permitted.

The source address C(R3) depends only on the bottom four bits of R3 under the 32-bit model. It can therefore take only 16 distinct values. This arithmetic property does not establish that those addresses are valid.

### 6. Intervening calls

After the first byte write, F-U31 is invoked with:

1. W(R6 shifted left by two).
2. The same W(R6 shifted left by two).
3. W(R6 shifted left by four).

F-U32 is then invoked with no explicit arguments shown, producing R7. F-U33 receives R7 as its explicit argument.

Next, F-U34, F-U35, and F-U36 are invoked with no explicit arguments shown, followed by F-U37, which produces R8.

R6 and R3 remain the captured values used for the next direct byte operations. This does not imply that the intervening dependencies preserve any memory or global state.

### 7. Second byte-directed write

The routine reads one unsigned byte from C(R3) **again**. Denote the second byte **Q2**.

It writes the low eight bits of:

(R6 AND 1,073,741,823) shifted right by 20

to address Q2 + 17.

The mask retains bits 0 through 29. The subsequent shift produces a value of up to ten bits, but the byte write keeps only its bottom eight bits. Consequently, the observable stored byte is exactly bits 20 through 27 of R6; bits 28 and 29 do not survive the store.

The second source read is required by the represented behavior. Q2 must not be replaced by Q1: intervening dependencies may change the source memory.

### 8. Final calls and terminal boundary

After the second byte write, the routine invokes:

1. F-U38 with W(R8 shifted left by two).
2. F-U39, F-U40, and F-U41 with no explicit arguments shown.
3. F-U42 with no explicit arguments shown, capturing R9.
4. F-U43 with W(R9 shifted left by two) and Q2, in that order.
5. F-U44 with no explicit arguments shown.

F-U43 receives the previously captured Q2. No third read of C(R3) is shown.

If F-U44 returns, the visible representation reaches the invalid-instruction boundary. No later behavior is established.

## Side effects

The direct memory effects, in order, are:

| Order | Access |
|---|---|
| 1 | Write P as a word to A(P) |
| 2 | Write P as a word to A(R1) |
| 3 | Write W(4R2) as a word to B(R2) |
| 4 | Read unsigned word V from A(R2) |
| 5 | Write V shifted right by 12 as a word to A(R3) |
| 6 | Write W(4R4) as a word to B(R4) |
| 7 | Write W(4R4) as a word to A(R5) |
| 8 | Read byte Q1 from C(R3) |
| 9 | Write the first derived byte to Q1 + 17 |
| 10 | Read byte Q2 from C(R3) |
| 11 | Write the second derived byte to Q2 + 17 |

The listed accesses are interleaved with the dependency calls described above. They are not one uninterrupted memory transaction.

Aliasing is unrestricted by the visible interface. Word destinations can coincide with other word destinations or later read locations. Byte destinations can coincide with each other or overlap earlier word storage. A later successful write can therefore overwrite earlier effects.

No allocation, release, ownership transfer, reference-count operation, lock, or rollback is explicitly identified. A pointer-like integer passed to a dependency does not prove a transfer of ownership.

No local error status is checked. If an access faults or a dependency does not return, the subsequent operations are not reached. There is no represented cleanup or reversal of already completed effects.

## Dependencies

All 44 call sites are indirect. The first target depends on incoming execution context; the others are represented using fixed code-location arithmetic. Their implementations and authoritative identities are unavailable.

| Opaque call-site IDs | Explicit interface visible here |
|---|---|
| F-U01 | Receives P |
| F-U02–F-U05 | No explicit arguments shown; F-U05 yields R1 |
| F-U06 | Receives W(4R1) |
| F-U07–F-U10 | No explicit arguments shown; F-U10 yields R2 |
| F-U11 | Receives W(4R2) |
| F-U12–F-U15 | No explicit arguments shown; F-U15 yields R3 |
| F-U16 | Receives W(4R3) |
| F-U17–F-U20 | No explicit arguments shown; F-U20 yields R4 |
| F-U21 | Receives the three values specified in phase 4 |
| F-U22–F-U25 | No explicit arguments shown; F-U25 yields R5 |
| F-U26 | Receives W(4R5) |
| F-U27–F-U30 | No explicit arguments shown; F-U30 yields R6 |
| F-U31 | Receives two copies of W(R6 shifted left by two), then W(R6 shifted left by four) |
| F-U32 | No explicit arguments shown; yields R7 |
| F-U33 | Receives R7 |
| F-U34–F-U37 | No explicit arguments shown; F-U37 yields R8 |
| F-U38 | Receives W(R8 shifted left by two) |
| F-U39–F-U42 | No explicit arguments shown; F-U42 yields R9 |
| F-U43 | Receives W(R9 shifted left by two), then Q2 |
| F-U44 | No explicit arguments shown |

“No explicit arguments shown” is not evidence of a verified zero-argument ABI. Hidden register inputs, shared-state requirements, memory effects, ownership behavior, and exceptional exits are all **documented_only**.

## Acceptance examples

These are conditional static expectations, not executed tests. Each assumes the necessary addresses are accessible, the stated dependency results occur, and earlier calls return.

1. **Initial observable write.** With P equal to 3, the first direct effect is a word write of 3 to address 32. F-U01 is entered afterward with argument 3.

2. **Read-after-write alias.** With R2 equal to zero, both B(R2) and A(R2) equal 20. The routine writes zero there and then reads V from the same address. With ordinary memory and no intervening external modification, V is zero.

3. **Unsigned word shift.** If V is hexadecimal F1234567, the later word written to A(R3) is hexadecimal 000F1234. Sign extension would not satisfy this behavior.

4. **Field arguments.** With R3 equal to 15, F-U21 receives 983,040 as its first argument and hexadecimal C0000000 as its third. Its second argument still depends independently on R4.

5. **Distinct byte destinations.** With R6 equal to hexadecimal 0AB00003, Q1 equal to 5, and Q2 equal to 9, the first byte write stores 12 at address 22; the second stores hexadecimal AB at address 26. Reusing Q1 for the second destination would be incorrect.

6. **Repeated destination.** If Q1 and Q2 both equal 5, both byte writes target address 22. The second write occurs later, after the intervening calls. Assuming ordinary memory and no later external modification, it replaces the first byte.

7. **Captured second byte.** If Q2 equals 9 and a later dependency changes the byte at C(R3), F-U43 still receives 9 as its second explicit argument.

8. **Partial execution on failure.** If F-U01 does not return, the initial word write has already occurred, but no subsequent direct write is reached. No rollback is represented.

## Uncertainties

The following are **documented_only**, not guessed behavior:

- The actual application purpose and the meaning of the computed addresses.
- Instruction set, ABI, endianness, alignment enforcement, and verified arithmetic widths.
- Reliability of the recovered control flow, especially near the invalid-instruction boundary.
- Dependency identities, signatures, global-state effects, memory ownership, and return guarantees.
- Whether addressed memory is ordinary RAM, device memory, protected storage, or unmapped.
- Whether concurrent or device-driven changes can occur between accesses.
- Exact fault delivery and whether the terminal boundary corresponds to an executable invalid instruction or an incomplete static representation.
- Whether any normal continuation exists outside the supplied representation.

The 32-bit modular model makes the listed arithmetic precise without asserting portable signed-language behavior. Any architecture evidence contradicting that model would require revisiting the affected address, shift, and truncation clauses.

## Evidence

The supplied static material supports one explicit integer input, 44 indirect call sites, six direct word writes, one direct word read, two unsigned byte reads, two byte writes, and an invalid-instruction terminal boundary. The order, widths, constants, masks, shifts, and consumed call results above derive from those represented operations.

The warning about invalid instruction data limits confidence in control-flow completeness. It is not evidence of an application-level error policy.

No implementation access, dependency inspection, instruction-level validation, or runtime tests have been performed. The acceptance examples are arithmetic consequences of the stated model, not observed executions.
