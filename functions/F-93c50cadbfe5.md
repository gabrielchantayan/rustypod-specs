<!-- Trusted provenance: F-93c50cadbfe5; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-93c50cadbfe5

## Purpose

The visible routine performs an ordered sequence of indirect calls and computed memory writes, then enters an unconditional repeating sequence of calls and byte clears.

No application-level purpose is established. The evidence does not identify the accessed locations as objects, allocated storage, device registers, or tables. It also does not establish that dependency results represent pointers, handles, or owned resources.

There is no normal return on the displayed path. Reaching and continuing the repeating region requires valid memory accesses, callable targets, and dependencies that return normally.

This is a conditional contract for the supplied static representation, not a verified executable specification. The representation reports invalid instruction data and removal of unreachable regions. Those limitations remain relevant to every control-flow claim below.

## Interface

### Explicit inputs

The apparent interface has four argument positions and no declared result.

| Position | Contract name | Width and visible use |
|---|---|---|
| First | A | Unsigned 32-bit value. Stored directly, used in address arithmetic, passed to a dependency, and masked to select a byte-read location. |
| Second | B | Four-byte value. No visible use. |
| Third | C | Unsigned 32-bit value. Its logical right shift by seven is passed to a dependency. |
| Fourth | D | Unsigned 32-bit value. Stored directly and used to construct a byte-write address and a dependency argument. |

An additional incoming machine-state value selects the first indirect call target after addition of `0xEE6`. Its ABI role, origin, and validity requirements are unknown. It is not established as a fifth explicit argument.

### Outputs and completion

The visible effects are memory accesses and dependency invocations. No return value, successful completion status, or normal exit is defined.

No guarantee is made that execution reaches the loop. A failed access, invalid target, non-returning dependency, or inaccurate recovered control flow can prevent later operations.

### Arithmetic and access conventions

The following notation describes apparent 32-bit machine behavior:

- `W(x)` is the low 32 bits of x.
- `L8(x)` is the low eight bits of x.
- Unsigned right shifts are logical.
- A word store transfers 32 bits.
- A byte read or store transfers eight bits.
- A selector-byte read produces an unsigned value from 0 through 255.
- Addresses are numeric address expressions, not object-relative offsets.

Some intermediate values and dependency results have signed integer types in the supplied representation. Modular formulas below describe the apparent 32-bit bit-pattern interpretation; they do not establish that signed overflow or signed left shift would be valid in a portable source language.

Byte order, alignment requirements, and access atomicity are unknown.

### Preconditions for continued execution

Each access must be permitted by the execution environment, each target must support the displayed invocation, and each dependency must return for the next operation to occur.

No bounds check, null check, target validation, allocation check, or error-recovery branch is visible.

## Behavior

### Dependency naming and ordering

F-D001 through F-D060 are opaque **call-occurrence IDs**, not verified function identities. Distinct IDs may resolve to the same function. A repeated ID denotes a repeated call position, not necessarily a constant target.

In particular, **F-D051 selects a different target on its first invocation than on later iterations**. Its target transition is specified below.

A range of calls denotes separate invocations in ascending ID order. “No explicit arguments” describes the displayed invocation, not an independently verified ABI signature.

Dependencies may modify memory or globals. Selector contents are therefore determined by each actual read, not assumed constant across calls. No dependency is assumed to allocate, release, preserve, or take ownership of a resource.

### 1. Initial word store and calls

Before any dependency invocation, store A as a 32-bit word at:

`W(4 × A + 20)`.

Then perform:

| Call occurrence | Explicit arguments | Result use |
|---|---|---|
| F-D001 | A | Ignored. |
| F-D002 | None | Retained as a four-byte value. |
| F-D003 | Result of F-D002 | Ignored. |
| F-D004 | None | Retained as a four-byte value. |
| F-D005 | Result of F-D004 | Ignored. |
| F-D006–F-D008 | None | Ignored individually. |
| F-D009 | None | Retained as R. |

Store D as a 32-bit word at `W(4 × R + 20)`.

Invoke F-D010 with `W(4 × R)`. Invoke F-D011, F-D012, and F-D013 without explicit arguments, ignoring their results. Invoke F-D014 without explicit arguments and retain its result as S.

### 2. Two ordered stores to one scaled location

Define `T = W(16 × S)`.

Store `W(4 × S)` as a 32-bit word at `W(T + 20)`.

Invoke F-D015 with these three arguments, in order:

1. T;
2. T;
3. C logically shifted right by seven.

The first two arguments have identical bit patterns.

Invoke F-D016, F-D017, and F-D018 without explicit arguments, ignoring their results. Invoke F-D019 without explicit arguments and retain its result as U.

Store T as a 32-bit word at `W(T + 20)`. This is the same computed location used before F-D015. Its address is derived from S, not from U. Intervening dependencies may also have accessed or changed that location.

Invoke F-D020 with `W(4 × U)`. Invoke F-D021, F-D022, and F-D023 without explicit arguments, ignoring their results. Invoke F-D024 without explicit arguments and retain its unsigned 32-bit result as V.

### 3. First selected byte store

Define the retained selector index:

`H = (A AND 0x7FFFFFFF) logically shifted right by 7`.

H ranges from zero through `0x00FFFFFF`. Bit 31 and bits 0 through 6 of A do not contribute to H. The selector-read address `H + 1` ranges from 1 through `0x01000000`.

Read an unsigned byte from `H + 1`. If that read yields b, store `L8(4 × V)` at address `b + 17`.

The selected destination lies between 17 and 272 inclusive. The byte read from `H + 1` supplies an address selector; the store is not inherently directed back to `H + 1`.

Then perform:

| Call occurrence | Explicit arguments | Result use |
|---|---|---|
| F-D025 | `W(4 × V)`, repeated three times | Ignored. |
| F-D026 | None | Retained as a four-byte value. |
| F-D027 | Result of F-D026 | Ignored. |
| F-D028–F-D030 | None | Ignored individually. |
| F-D031 | None | Retained as J. |

### 4. Second selected byte store

Read the unsigned byte at `H + 1` again. The earlier selector value is not reused.

For the newly read value b, store the following byte at `b + 17`:

`L8((V AND 0x3FFFFFFF) logically shifted right by 20)`.

The masked-and-shifted expression can contain ten bits, but the byte store retains only eight. The stored value therefore consists of bits 20 through 27 of V. Bits 28 and 29 do not survive the store.

Invoke F-D032 with `W(4 × J)`. Invoke F-D033, F-D034, and F-D035 without explicit arguments, ignoring their results. Invoke F-D036 without explicit arguments and retain its unsigned 32-bit result as K.

### 5. Direct byte store and two selected clears

Store `L8(4 × K)` at:

`(D logically shifted right by 7) + 28`.

This destination ranges from 28 through `0x0200001B`; the addition does not overflow 32 bits.

After that store, read the unsigned selector byte at `H + 1` and store zero at the resulting address `b + 17`.

These operations are ordered. If the direct store changes the selector location, the subsequent read uses the post-store contents, subject to the environment’s memory semantics.

Next perform:

| Call occurrence | Explicit arguments | Result use |
|---|---|---|
| F-D037 | D logically shifted right by seven | Ignored. |
| F-D038 | None | Ignored. |
| F-D039 | `(K AND 0x3FFFFFFF)` logically shifted right by six | Ignored. |
| F-D040–F-D042 | None | Ignored individually. |
| F-D043 | None | Retained as N. |

Read the unsigned byte at `H + 1` again and store zero at its newly selected destination `b + 17`.

Invoke F-D044 with `W(4 × N)`. Invoke F-D045, F-D046, and F-D047 without explicit arguments, ignoring their results. Invoke F-D048 without explicit arguments and retain its unsigned 32-bit result as P.

### 6. Preparation for repetition

Define:

- `Q = (P AND 0x3FFFFFFF) logically shifted right by 25`;
- `E = W(4096 × K + 1)`.

Initially Q ranges from 0 through 31. E derives from K, not from H or P. Its formula follows the displayed scaling of the retained low-32-bit value `4 × K` by another factor of 1024.

Read an unsigned byte from E and store zero at the address equal to that byte plus 17.

Invoke F-D049 with Q shifted left by one. Its argument ranges from 0 through 62.

Invoke F-D050 without explicit arguments and retain its 32-bit result as the initial loop state.

The control-base value retained for the first F-D051 invocation is `0x08832789`.

### 7. Unconditional repeating sequence

Each iteration has the following displayed order:

| Step | Operation |
|---|---|
| 1 | Invoke F-D051 with the current loop state and Q logically shifted right by eight, using the target-selection rule below. |
| 2 | Invoke F-D052, F-D053, and F-D054 without explicit arguments; ignore their results. |
| 3 | Invoke F-D055 without explicit arguments and retain its result as X. |
| 4 | Read the unsigned byte at E, then store zero at the selected address `byte + 17`. |
| 5 | Invoke F-D056 with `W(4 × X)`. |
| 6 | Invoke F-D057, F-D058, and F-D059 without explicit arguments; ignore their results. |
| 7 | Set the retained control base for the next F-D051 invocation to `0x088327C7`, immediately before invoking F-D060. Invoke F-D060 without explicit arguments and retain its result as Y. |
| 8 | Set the loop state to `W(4 × Y)` and set Q to `W(Y shifted left by 26)`. |
| 9 | Repeat unconditionally. |

**F-D051 target selection:**

- On the first iteration, its displayed target is `0x08832789 + 0xFFE`, giving `0x08833787`.
- On subsequent iterations, its displayed target is `0x088327C7 + 0xFFE`, giving `0x088337C5`.
- The later base is assigned immediately before F-D060 and remains the base used at the next loop entry. It is not reset to the first-iteration base.
- These are target-address expressions in the representation. Instruction-set mode conventions, target validity, and actual callable identities remain unknown.

The first F-D051 invocation receives zero as its second argument because the initial Q is at most 31.

On later iterations, that argument equals:

`(Y AND 63) shifted left by 18`,

where Y is the preceding iteration’s F-D060 result. It need not be zero.

E remains derived from the pre-loop K. Each clear rereads the byte at E; neither its contents nor the selected destination is cached by the visible body.

## Side effects

### Explicit memory operations

Before the loop, the visible body performs exactly:

| Access category | Count | Breakdown |
|---|---:|---|
| 32-bit stores | 4 | Initial A store; D store; two stores at `W(T + 20)`. |
| Byte stores | 6 | Two selected value stores; one direct value store; two clears selected through `H + 1`; one clear selected through E. |
| Unsigned selector-byte reads | 5 | Four reads at `H + 1`; one read at E. |

Each completed loop iteration adds one unsigned selector-byte read at E and one selected zero-byte store.

These counts exclude all accesses performed by dependencies. They count access occurrences, not distinct addresses. There are no explicit word reads in the visible body.

### Ordering and aliasing

The contract preserves displayed call and access order. It does not establish hardware barriers, volatile semantics, cross-thread synchronization, or atomicity.

No non-aliasing guarantee is visible. Word destinations, direct byte destinations, selector locations, and selected byte destinations may overlap where their formulas permit. A completed store or an intervening dependency can therefore affect a later read.

Repeated reads of one selector expression are separate observations. Their resulting stores need not use the same destination.

### Ownership and errors

No allocation, release, ownership transfer, or resource-lifetime protocol is established.

Forwarding a four-byte result does not establish that it is an owned object or a borrowed reference. Ignoring a result does not establish that the associated call lacks side effects.

No status inspection, cleanup branch, retry policy, or error return is visible. Faults, traps, dependency exceptions, and non-returning dependencies have documented_only outcomes: the supplied representation does not specify what happens beyond the interrupted path.

## Dependencies

The visible call order is F-D001 through F-D050 once, followed by F-D051 through F-D060 repeatedly.

F-D001 uses incoming machine state plus `0xEE6` for target selection. Other targets are expressed through control-address arithmetic. F-D051 specifically changes from its entry-base target to its backedge-base target after the first iteration.

For every dependency:

- Its semantic purpose and callable identity are unknown.
- Its displayed arguments are not an independently verified ABI declaration.
- Results have only the widths and uses described here.
- Global-memory preservation is not assumed.
- Allocation, release, or ownership behavior is not assumed.
- Normal return is required to observe the following operation.

The two target expressions associated with F-D051 are distinct. Whether they designate different semantic functions, alternate entry points, or something else is documented_only.

## Acceptance examples

These examples are conditional arithmetic and ordering consequences of the representation, not executed tests. Each assumes the required accesses succeed and preceding dependencies return.

1. **Initial store before calls.**  
   With A equal to 3, the initial operation stores the 32-bit value 3 at address 32. F-D001 subsequently receives 3.

2. **The scaled location is reused independently of U.**  
   If F-D014 returns 2, the routine stores 8 at address 52. F-D015 receives 32, 32, and C logically shifted right by seven. After F-D019 returns, the routine stores 32 at address 52, irrespective of U.

3. **Masking does not erase other uses of A.**  
   A equal to zero and A equal to `0x80000000` both give H equal to zero and an initial store address of 20 under the 32-bit interpretation. Their stored values and F-D001 arguments nevertheless differ.

4. **Selected stores truncate to bytes.**  
   If V is `0x12345678`, the first selected value store writes `0xE0`; the second writes `0x23`. If both relevant selector reads yield 5, both destinations are address 22. That shared destination is conditional on the separate reads yielding the same value.

5. **A direct store can change the following selector.**  
   With A equal to `0xD80`, H is 27 and the selector location is address 28. With D equal to zero, the direct byte destination is also 28. If K equals 1, that store writes 4 at address 28; under ordinary sequential byte-memory semantics and without an intervening external change, the immediately following selector read yields 4 and the clear targets address 21.

6. **Loop entry changes both target selection and arguments.**  
   If P is `0x3FFFFFFF`, F-D049 receives 62. The first F-D051 call uses displayed target `0x08833787`, takes the F-D050 result as its first argument, and receives zero as its second argument. If F-D060 then returns 3, the next F-D051 call instead uses displayed target `0x088337C5`, with arguments 12 and `0x000C0000`.

7. **Zero does not terminate repetition.**  
   If F-D060 returns zero, the next loop state and Q are zero. Another iteration follows, using the subsequent-iteration F-D051 target.

8. **Pre-loop access accounting.**  
   A path reaching the first F-D051 invocation has completed four explicit word stores, six explicit byte stores, and five explicit selector-byte reads. Completing one loop iteration adds one selector-byte read and one byte store.

## Uncertainties

### Established static evidence

The representation establishes the displayed argument uses, access widths, ordering, retained results, selector rereads, four pre-loop word stores, six pre-loop byte stores, five pre-loop selector reads, and unconditional repeating structure.

It also establishes the control-base transition governing F-D051: the first invocation uses the base retained before the loop, while later invocations use the different base assigned immediately before F-D060.

### Inferred intent

No application-level intent is adopted. Scaling, masking, small selected destinations, and repeated clears may have domain-specific meanings, but the evidence does not identify them.

### Documented-only unknowns

- Whether the recovered path corresponds to valid executable instructions.
- Whether removed regions affect actual execution.
- Whether the apparent four-argument, no-result interface is accurate.
- Dependency ABIs, callable identities, effects, and resource behavior.
- The origin and meaning of incoming state used by F-D001.
- Address-space layout, permissions, alignment rules, and byte order.
- Exact executable semantics of operations displayed with signed intermediate types.
- Instruction-set interpretation of computed call targets.
- Fault, exception, trap, and non-return behavior.
- Concurrency, memory-ordering, and device-specific semantics.

## Evidence

This contract is based solely on the supplied static representation. Its invalid-instruction warning and notices of removed unreachable regions limit confidence in recovered executable control flow.

The memory counts follow the explicit access occurrences: four word stores; two selected byte-value stores; one direct byte-value store; three selected clears; and five selector-byte reads before repetition.

The F-D051 target transition follows the distinct control-base assignments before loop entry and immediately before F-D060.

No runtime tests, dependency inspections, memory observations, or independent ABI verification have been performed. The acceptance examples are conditional static deductions, not observed executions.
