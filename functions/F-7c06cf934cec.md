<!-- Trusted provenance: F-7c06cf934cec; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-7c06cf934cec

## Purpose

F-7c06cf934cec performs an ordered sequence of indirect function calls and memory writes, then enters a loop with no visible exit.

The supplied static representation establishes the sequence and several data transformations. It does **not** establish an application-level purpose, a normal return path, a recoverable error protocol, or the meaning of the affected memory. In particular, the low numerical addresses used by several byte writes must not be presumed to represent ordinary writable objects.

This contract describes the visible path, conditional on each preceding operation completing and each invoked dependency returning. The supplied representation carries control-flow reliability warnings; consequently, this is a source-facing behavioral contract, not proof that the recovered path matches an executable implementation in every detail.

## Interface

### Explicit inputs

| Input | Represented width and interpretation | Visible use |
|---|---|---|
| First argument | 32-bit signed integer | Scaled by four; used both as stored data and to derive an address. |
| Second argument | Opaque 32-bit value | No explicit use on the displayed path. |
| Third argument | Opaque 32-bit value | No explicit use on the displayed path. |
| Fourth argument | 32-bit unsigned integer | Logically shifted right by seven to form a call argument and a byte-write address. |

“No explicit use” does not establish that a dependency cannot observe the second or third argument through an undocumented calling convention or machine state.

### Additional entry state

The visible behavior also depends on two values outside the declared argument list:

- An unsigned 32-bit entry-register value. Its logical right shift by eight produces a saved selector-source base.
- An entry link-register value. Adding `0xFFE` determines the first indirect call target.

Their initialization, validity, preservation rules, and relationship to the explicit arguments are unknown. A caller supplying only the four declared arguments has not necessarily satisfied the actual entry requirements.

### Outputs and termination

The declared return type provides no result.

The displayed path does not reach a return. After a finite prefix, execution enters an unconditional loop. If dependencies return and memory accesses succeed indefinitely, the function continues indefinitely.

A dependency may fail, trap, transfer control elsewhere, or never return; its behavior is not supplied. Such an event prevents any later operation in this contract from being guaranteed.

### Width and arithmetic conventions

The representation uses 8-bit byte accesses and 32-bit integer values and word accesses.

For the formulas below:

- **Scaled input** means the first argument multiplied by four.
- **Selector base** means the unsigned entry-register value logically shifted right by eight.
- **Fourth-derived value** means the fourth argument logically shifted right by seven.
- **Low byte** means the least significant eight bits of a value.
- Unsigned right shifts are logical shifts.

Unsigned 32-bit left shifts retain the low 32 bits. Some other calculations are represented using signed integers, including the initial multiplication, some returned-value shifts, and a later multiplication by `0x400`. Signed overflow, negative signed left shifts, address overflow, and the exact machine interpretation of those expressions are not resolved by the supplied material. Modular bit-pattern interpretations of those operations are conditional, not unconditional language-level guarantees.

## Behavior

### 1. Initial word write and call group

The first visible memory operation writes the first argument as a 32-bit word to the address:

**scaled input + `0x14`**

There is no preceding visible validation or memory read.

The function then captures the selector base and makes five calls in order:

| Order | Dependency | Explicitly shown arguments | Result used locally |
|---|---|---|---|
| 1 | F-C01 | Scaled input | None |
| 2 | F-C02 | None | None |
| 3 | F-C03 | None | None |
| 4 | F-C04 | None | None |
| 5 | F-C05 | None | First intermediate result |

F-C01 uses the entry-state-derived target. The remaining targets are derived from fixed values in the supplied representation.

After F-C05 returns, the function writes the scaled input as a 32-bit word to the **same previously derived address**. Thus, absent intervening aliasing or dependency effects, that location first receives the original first argument and later receives its fourfold scaling.

Next:

| Order | Dependency | Explicitly shown arguments | Result used locally |
|---|---|---|---|
| 6 | F-C06 | First intermediate result shifted left by two | None |
| 7 | F-C07 | None | None |
| 8 | F-C08 | None | None |
| 9 | F-C09 | None | None |
| 10 | F-C10 | None | First unsigned result |

Dependencies between the two word writes may modify memory. The second write is still visibly attempted; preservation of the first write is not assumed.

### 2. First selector-mediated byte writes

A selector-mediated write consists of two ordered accesses:

1. Read one unsigned byte at **selector base + 1**.
2. Write one byte at **the byte’s numerical value + `0x11`**.

The loaded byte is an address component, not a pointer-sized value. Therefore, the destination is numerically between `0x11` and `0x110`, inclusive, before considering platform address interpretation.

The function shifts the first unsigned result left by two and performs a selector-mediated write of its low byte.

It then makes these calls:

| Order | Dependency | Explicitly shown arguments | Result used locally |
|---|---|---|---|
| 11 | F-C11 | Three copies of the shifted first unsigned result | None |
| 12 | F-C12 | None | Opaque 32-bit result |
| 13 | F-C13 | That opaque result | None |
| 14 | F-C14 | None | None |
| 15 | F-C15 | None | None |
| 16 | F-C16 | None | None |
| 17 | F-C17 | None | Second intermediate result |

After these calls, it rereads the selector byte at **selector base + 1** and writes the low byte of:

**(first unsigned result AND `0x3FFFFFFF`) logically shifted right by 20**

The stored byte contains original result bits 20 through 27. The complete shifted expression can occupy ten bits, but the write retains only eight.

The selector read is fresh. Dependencies may have changed it, so this write need not target the same address as the preceding selector-mediated write.

### 3. Fourth-argument-derived write and further selector writes

The next calls are:

| Order | Dependency | Explicitly shown arguments | Result used locally |
|---|---|---|---|
| 18 | F-C18 | Second intermediate result shifted left by two | None |
| 19 | F-C19 | None | None |
| 20 | F-C20 | None | None |
| 21 | F-C21 | None | None |
| 22 | F-C22 | None | Second unsigned result |

The function shifts the second unsigned result left by two and retains the resulting 32-bit value for later address derivation.

It then performs, in order:

1. A one-byte write of the low byte of that shifted result to **fourth-derived value + `0x1C`**.
2. A fresh selector-byte read at **selector base + 1**, followed by a zero-byte write to the selected address.

The fourth-derived value ranges from zero through `0x01FFFFFF`; therefore, the direct byte-write address ranges from `0x1C` through `0x0200001B`.

Next:

| Order | Dependency | Explicitly shown arguments | Result used locally |
|---|---|---|---|
| 23 | F-C23 | Fourth-derived value | None |
| 24 | F-C24 | None | None |
| 25 | F-C25 | (Second unsigned result AND `0x3FFFFFFF`) logically shifted right by six | None |
| 26 | F-C26 | None | None |
| 27 | F-C27 | None | None |
| 28 | F-C28 | None | None |
| 29 | F-C29 | None | Third intermediate result |

The argument to F-C25 ranges from zero through `0x00FFFFFF`.

After F-C29 returns, the function performs another fresh selector-mediated zero-byte write using the original selector base.

It continues:

| Order | Dependency | Explicitly shown arguments | Result used locally |
|---|---|---|---|
| 30 | F-C30 | Third intermediate result shifted left by two | None |
| 31 | F-C31 | None | None |
| 32 | F-C32 | None | None |
| 33 | F-C33 | None | None |
| 34 | F-C34 | None | Third unsigned result |

The function derives a five-bit value:

**(third unsigned result AND `0x3FFFFFFF`) logically shifted right by 25**

Its range is zero through 31.

### 4. Alternate selector source and loop entry

A second selector-source address is formed from:

**the retained shifted second unsigned result × `0x400` + 1**

The signed-arithmetic caveat applies to this multiplication. Under a 32-bit modular machine interpretation, the address is equivalent to the second unsigned result shifted left by twelve, retaining 32 bits, then increased by one.

The function:

1. Reads an unsigned byte from this alternate selector-source address.
2. Writes zero to the byte address obtained by adding `0x11` to the loaded byte.
3. Calls F-C35 with the five-bit value shifted left by one.
4. Calls F-C36 with no explicit arguments, retaining its result as the initial loop-carried first value.

The argument to F-C35 is even and ranges from zero through 62.

### 5. Unconditional loop

The first iteration begins with F-C37. Every subsequent iteration begins with F-C47 instead. This target transition is visible and must not be collapsed into a single invariant call target.

Each iteration performs the following ordered work:

1. Call its iteration-opening dependency with two explicit arguments:
   - The current loop-carried first value.
   - The current loop-carried unsigned value logically shifted right by eight.
2. Call F-C38, F-C39, F-C40, then F-C41, with no explicit arguments. Retain the result of F-C41.
3. Reread the byte at the alternate selector-source address and write zero to the address selected by that byte plus `0x11`.
4. Call F-C42 with the F-C41 result shifted left by two.
5. Call F-C43, F-C44, F-C45, then F-C46, with no explicit arguments. Retain the result of F-C46.
6. Set the next loop-carried first value to the F-C46 result shifted left by two.
7. Set the next loop-carried unsigned value to the F-C46 result shifted left by 26.
8. Continue unconditionally, using F-C47 as the next opening dependency.

For the first iteration, the loop-carried unsigned value is the five-bit value derived before loop entry. Consequently, the second argument to F-C37 is always zero.

Under the 32-bit bit-pattern interpretation of the later left shift, subsequent second arguments equal:

**the low six bits of the preceding F-C46 result, shifted left by 18**

They range from zero through `0x00FC0000` in multiples of `0x00040000`.

There is no visible loop exit condition, counter limit, return, or cleanup operation.

## Side effects

### Direct memory effects

Before loop entry, the visible path performs:

- Two 32-bit writes to the first-argument-derived address.
- Four byte reads from the original selector-source address, each followed by one selected byte write.
- One direct byte write using the fourth argument.
- One byte read from the alternate selector-source address, followed by a selected zero-byte write.

Each complete loop iteration adds one alternate selector-byte read and one selected zero-byte write.

The four original-selector writes carry, in order:

1. The low byte of the first unsigned result shifted left by two.
2. The byte extracted from bits 20 through 27 of that result.
3. Zero.
4. Zero.

Every selector-byte load is repeated at its stated point. Neither selector bytes nor their resulting destination addresses are cached across calls.

The listed addresses may alias each other, dependency-managed memory, or the selector sources themselves. Such aliasing may affect later reads. No independence between these regions is established.

### Ownership, state preservation, and errors

No direct allocation, release, ownership transfer, or resource acquisition is visible. This does not establish that dependencies perform none.

No explicit global-state preservation contract is available for dependencies. Their memory effects, register effects, synchronization behavior, and ownership behavior remain unknown.

No visible bounds check, null check, alignment check, status check, retry policy, or error return precedes the accesses and calls. Successful progression requires valid executable targets and accessible memory with the necessary permissions. Alignment and device-memory requirements depend on the execution environment.

The order above describes the visible execution sequence; it does not establish atomicity, inter-thread synchronization, or hardware memory-barrier semantics.

## Dependencies

F-C01 through F-C47 are document-local opaque function IDs for unresolved call sites. They are not recovered symbol identities. Separate IDs do not prove separate underlying functions or prohibit target aliasing.

- F-C01 has an entry-state-dependent target.
- F-C16 has a target derived from an embedded code reference.
- The remaining targets are fixed-derived in the supplied representation.
- F-C37 is used only for the first loop opening.
- F-C47 is used for later loop openings.

The behavior tables specify only arguments explicitly displayed and results subsequently consumed. “None” in an argument column means no explicit arguments are shown; it does not prove that the actual calling convention accepts no additional register or stack inputs.

No dependency implementation or independent dependency contract was supplied. In particular, ignored results are not evidence of successful execution.

## Acceptance examples

These are conditional specification examples, not executed tests.

1. **Initial address and overwrite.**  
   With first argument 3, accessible memory, and returning dependencies, the first direct word write stores 3 at address `0x20`. After F-C05, the second direct word write stores 12 at that same address.

2. **Selector changes across calls.**  
   If the original selector byte is 2 at its first read and 9 at its second read, the corresponding selected writes target `0x13` and `0x1A`. A consumer must not require both writes to use the first destination.

3. **Byte truncation.**  
   If F-C10 returns `0x12345678`, the first selected byte write stores `0xE0`; the second stores `0x23`, assuming execution reaches both writes. These are eight-bit writes, not writes of the complete shifted expressions.

4. **Fourth-argument addressing.**  
   If the fourth argument is `0x100`, its derived value is 2. The direct byte-write address is `0x1E`, and the explicit argument to F-C23 is 2.

5. **Masked high bits.**  
   If F-C22 returns `0xFFFFFFFF`, the direct byte write stores `0xFC`, and F-C25 receives `0x00FFFFFF`. These unsigned transformations do not resolve the separate signed-overflow question in the later alternate-selector address calculation.

6. **First loop argument bound.**  
   If F-C34 returns `0xFFFFFFFF`, the five-bit value is 31. F-C35 receives 62, and F-C37 receives zero as its second explicit argument.

7. **Later loop arguments.**  
   If F-C46 returns 3 in an iteration, the next opening call is F-C47 with first argument 12 and second argument `0x000C0000`.

8. **Interrupted execution.**  
   If F-C12 does not return, F-C13 and all later direct operations are not reached. Earlier writes are not visibly rolled back.

## Uncertainties

The following are **documented_only**:

- The application-level purpose of the operation.
- The full calling convention and required entry-register state.
- Whether the displayed control flow faithfully represents executable behavior.
- Dependency identities, implementations, memory effects, ownership rules, and non-returning behavior.
- Signed-overflow and signed-shift semantics for the displayed expressions.
- Address-space mapping, alignment requirements, endianness of word writes, and fault handling.
- Whether low-address byte writes represent device access, ordinary memory, corruption, or reconstruction artifacts.
- Any externally imposed termination mechanism or concurrency protocol.

An interpretation as a repeated state-update or device-control operation is possible, but is inferred intent only and is not part of the established contract.

## Evidence

The static evidence is the supplied function representation: four explicit parameters, additional entry-state reads, ordered indirect calls, direct word and byte accesses, masks and shifts, and an unconditional final loop.

The supplied material also reports bad instruction data and removal of unreachable blocks. Those warnings limit confidence in control-flow completeness and recovered type accuracy; they do not justify inventing missing behavior.

No implementation access, dependency inspection, runtime execution, smoke run, or tests were performed. All acceptance examples are arithmetic and sequencing consequences of the displayed path, subject to the stated preconditions and uncertainties.
