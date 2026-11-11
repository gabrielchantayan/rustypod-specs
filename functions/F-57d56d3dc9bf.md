<!-- Trusted provenance: F-57d56d3dc9bf; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-57d56d3dc9bf

## Purpose

F-57d56d3dc9bf performs ordered memory accesses and indirect invocations along two paths: a terminal path with a represented return, and a repeating path with dependency-controlled addresses and a possible software breakpoint.

No application-level purpose is established. Interrupt- and exception-associated storage suggests low-level environment manipulation, but does not prove hardware configuration, a valid execution environment, or meaningful application behavior.

This contract describes the supplied static representation. It does not establish that all represented arithmetic has defined language-level semantics, that all paths are reachable, or that the apparent interface captures every required incoming value.

## Interface

### Explicit inputs and output

| Input | Represented width | Role |
|---|---|---|
| A | Signed 32 bits | Initial address component and halfword payload; replaced with zero by setup. |
| B | 32 bits | No explicit consumption is represented. |
| C | Unsigned 32 bits | Address scaling, masking, initialization, and payload generation; replaced by setup. |
| D | Unsigned 32 bits | Selects setup and supplies a shifted setup argument. |
| E | Signed 32 bits | Contributes to a repeating-path address and payload calculation. |

The represented return carries no value. No status, error object, or resource is returned. Only the terminal path has a represented normal return, conditional on its final dependency returning.

### Additional incoming state

Behavior also depends on:

- A signed 32-bit **scale**.
- A 32-bit address-valued **cursor**.
- A 16-bit **setup payload**.
- An unsigned 32-bit **data word**.
- A signed 32-bit **terminal target base**.
- An unsigned 32-bit **link value**.
- A secondary signed 32-bit result associated with the setup invocation.
- Mutable global storage, including address-bearing and exception-associated locations.

These roles describe consumed state, not a verified calling convention. Their initialization and transport are unresolved.

### Widths, arithmetic, and memory

Explicit computed-address accesses use 8-, 16-, 32-, and 64-bit widths. Addresses and stored pointer-shaped values are represented as 32 bits. All offsets in this contract are bytes.

Unsigned 32-bit operations wrap modulo \(2^{32}\). Narrow stores retain the payload’s low bits. A 64-bit result is described by its numerical low and high 32-bit halves; this does not establish memory endianness.

Signed multiplication, addition, and left shifts occur in address and payload calculations. Their overflow behavior is unresolved. Signed right shifts also require an execution-specific interpretation. In particular, the continuation expression includes a signed 32-bit right shift by 32; no portable result is established.

Formulas involving such operations specify represented dependencies, not unconditional wrapping behavior. Pointer arithmetic does not establish accessible storage or permission to cross object boundaries.

### Preconditions and ownership

No input validation, bounds checking, allocation, release, or ownership protocol is represented. Every reached access and invocation requires environmental support to continue. Addresses may be zero, low, unaligned, or overlapping.

Dependency-provided addresses are used without capacity checks. Neither receiving nor storing a pointer-shaped value establishes ownership or a release obligation.

## Behavior

### Setup and common entry effects

An address-bearing global is captured at entry. The initial **working base** is `0x000000a1`.

If D is zero, setup is skipped. Incoming scale, cursor, and data word remain relevant.

If D is nonzero, setup performs these effects in order:

1. Interrupt-associated storage receives `0x18`.
2. Another fixed global receives the setup payload.
3. A byte `0xce` is written at C plus a fresh observation of the first address-bearing global.
4. A separate global is read and converted to the unsigned 32-bit data word.
5. A control global is cleared.
6. F-opaque-01 is invoked with D logically shifted right by 18, `0xc0`, and the control global’s address.

The fresh observation in step 3 is distinct from the entry capture. Their values need not agree. The data-word source’s declaration is unavailable; a 16-bit read and zero-extension are not established.

After F-opaque-01:

- Cursor is derived from the entry-captured value shifted left once.
- C becomes `0x0897eca0`.
- A second address-bearing global is observed and multiplied by 32.
- A halfword `0xeca0` is written at four times the secondary setup result.
- Working base becomes zero.
- Another halfword `0xeca0` is written six bytes beyond the scaled second-global value.
- Scale becomes `0x10`, and A becomes zero.

The second-global observation occurs after the dependency. No dependency preservation of globals or referenced memory is assumed.

For the common effects, define **S** as scale multiplied by `0x02000000`, with the represented result converted to unsigned 32 bits, and **V** as the address derived from scale multiplied by `0x04000000`.

The ordered common stores are:

| Order | Destination | Width and payload |
|---:|---|---|
| 1 | A plus `0x30` | 16 bits: A’s low bits |
| 2 | S plus `0x40` | 32 bits: S |
| 3 | Working base | 8 bits: zero |
| 4 | Working base | 32 bits: data word |

The byte and word writes remain separate effects even if ordinary memory retains only the later word’s contents.

The terminal path is selected only when S has its high bit set and V is nonzero. Otherwise execution enters the repeating path.

Completed setup gives S = `0x20000000` and V = `0x40000000`, so its terminal predicate is false. These two particular products fit signed 32-bit range.

### Terminal path

A zero byte is first written at V. Define **T** as V displaced by the data word.

The following stores occur in order:

1. A halfword containing working base’s low bits at cursor plus two.
2. A 32-bit pointer-shaped value T at T plus `0x40`.
3. A halfword containing T’s low bits at T.
4. A self-addressed word whose destination and payload are scale multiplied by `0x40000000`, then increased by `0x11`.

The fourth calculation has signed-arithmetic uncertainty.

Let **intersection** be the unsigned 32-bit intersection of C and T. Shift it left once, interpret that result as signed 32 bits, and right-shift by ten to derive another address. Two consecutive halfword writes place that address’s low bits at the address itself. Both writes are represented. A halfword `0x50` is then written at intersection plus two.

Global and indirect memory effects follow:

1. Global storage identified at `0x000000fd` receives `0xf430`.
2. Reset-associated storage receives `0x0897f430`.
3. Undefined-instruction-associated storage receives zero.
4. Supervisor-associated storage is read and converted to an unsigned address.
5. A word `0x0897f430` is written through that address.

Missing global declarations prevent assigning physical widths to these global accesses. In particular, the supervisor-associated read is not established as a halfword read.

F-opaque-02 then receives:

| Position | Argument |
|---:|---|
| 1 | Supervisor-derived address plus 16 bytes, interpreted as signed and multiplied by `0x01000000` |
| 2 | Zero |
| 3 | `0x5000` |
| 4 | `0x56` |

The first argument retains signed-overflow uncertainty. If this represented invocation returns normally, F-57d56d3dc9bf returns without repeating-path initialization. Whether the underlying transfer is an ordinary returning call remains unresolved.

### Repeating-path initialization and header

Initialization establishes:

- Word `0x40` at working-base offset `0x28`.
- Working base’s own address at offset `0x2c`.
- An **accumulator** equal to data word bitwise-AND C.
- An **index** equal to \(8C-2\), using unsigned arithmetic.
- A word payload equal to cursor interpreted as signed, plus E, plus eight, stored at twice that sum.
- An **access base** equal to \(C \times \texttt{0x01000000}\).
- A word containing C at working-base offset four.
- The index’s low byte at access-base offset `0x14`.
- A **step** equal to \(C \times \texttt{0x02000000}\).
- A **header payload** equal to C shifted left by 28.
- An **accumulator destination** initially equal to access base.

The cursor/E-derived calculation uses signed arithmetic. The stores at working-base offsets `0x28` and `0x2c` precede that derived store; the C store and access-base byte follow it.

Each **header visit** performs:

1. A zero-byte store at access base.
2. An index-halfword store at access base.
3. A header-payload word store at access base.
4. Addition of step to accumulator, followed by a word store of the updated accumulator at accumulator destination.

Initially, all four writes share their starting address. On later header visits, access base is zero while accumulator destination retains a dependency-derived value; the initial aliasing must not be generalized.

### Dependency cycle and short restart

After a header, the outer dependency-cycle iteration initializes the next F-opaque-03 argument to zero. The inner cycle may subsequently continue with a different argument after a resumed breakpoint.

Before each F-opaque-03 invocation, step’s low byte is captured for possible later use.

F-opaque-03 returns 64 bits. Its low half supplies the destination of a halfword store containing that half’s low bits. Next, index multiplied by `0x40`, using unsigned arithmetic, supplies both destination and payload of a word store.

Cursor-related operations then occur in this order:

1. A word read through the current cursor becomes step.
2. A halfword containing the low bits of cursor plus four is written at index plus four.
3. Cursor is replaced by a word read at F-opaque-03’s high result half plus `0x50`.

Thus the old cursor supplies both the step-read address and the halfword payload before replacement. Earlier stores and dependency effects may affect these reads through aliasing.

F-opaque-04, F-opaque-05, F-opaque-06, and F-opaque-07 follow in order. The first two have no explicit arguments. F-opaque-06 receives step; F-opaque-07 receives step logically shifted right by three.

F-opaque-07’s high result half becomes access base. Its low half, interpreted as signed 32 bits and multiplied by 32, produces a value converted to unsigned 32 bits. That value supplies both destination and payload of a word store. Only afterward is it compared, unsigned, with `0x14`.

If the value is below `0x14`, execution leaves the inner cycle and starts another outer dependency-cycle iteration:

- No header stores or accumulator addition recur.
- The next F-opaque-03 argument is reset to **zero**.
- Its dynamic target base is `0x0897edf1`.

The intermediate value `0x0897eddf`, used in forming F-opaque-05’s target, is not passed to F-opaque-03 on this restart.

Under an explicitly assumed wrapping interpretation of the signed multiplication, the scaled value is a multiple of 32. Being below `0x14` then requires zero, making the preceding effect a word-zero store at address zero. This conclusion is conditional, not an unconditional signed-arithmetic guarantee.

### Extended cycle and continuation

A scaled value at least `0x14` enters the extended cycle:

1. Index is stored as a word at scaled-value offset `0x58`.
2. Cursor’s low half is stored at index offset `0x0e`.
3. Step becomes index shifted left by two.
4. Reset-associated storage receives a combined value with the previously captured byte in its low eight bits and a separately referenced three-byte component in its upper 24 bits.
5. F-opaque-08 receives cursor logically shifted right by two.
6. F-opaque-09 is invoked without explicit arguments.

The reset update establishes a value dependency, not a physical byte-only write or a guaranteed full-width read of the destination.

Let **L** and **U** be F-opaque-09’s low and high result halves. U becomes accumulator destination. The subsequent stores are ordered as follows:

| Order | Destination | Width and payload |
|---:|---|---|
| 1 | Twice L | 8 bits: captured byte |
| 2 | L plus eight | 8 bits: U’s low byte |
| 3 | Twice the previous accumulator | 32 bits: access base |
| 4 | L | 64 bits: entire dependency result |
| 5 | Access base | 32 bits: step |
| 6 | Access base plus four | 32 bits: new index |
| 7 | Access base plus eight | 32 bits: cursor |

Twice L uses signed multiplication after signed interpretation. Twice the previous accumulator uses unsigned arithmetic. The 64-bit store may overlap earlier stores and is not established as atomic.

The new index is cursor interpreted as signed and shifted left by eight. Accumulator becomes access base interpreted as signed and shifted left by two. Header payload becomes step multiplied by `0x100`, using unsigned arithmetic.

The continuation expression applies a signed left shift by 31 and signed right shift by 32 to cursor-derived state, then converts to unsigned and logically shifts right by 20. Its definite value is unresolved. If an environment supplies a 32-bit intermediate result, the final logical shift limits the condition to `0x000`–`0xfff`.

Access base is set to zero before the branch:

- **Condition zero:** execution revisits the header. Accumulator destination remains U, the dynamic target base is `0x0897ee1d`, and the next F-opaque-03 argument is zero.
- **Condition nonzero:** the condition is stored as a word at header-payload offset `0x20`, followed by a software breakpoint with immediate four. If execution resumes normally, index becomes signed U right-shifted by 18. The inner dependency cycle resumes without a header visit, using target base `0x08790d52` and signed header payload right-shifted by 29 as its next argument.

A subsequent below-threshold result still resets the following F-opaque-03 argument to zero through the outer iteration. There is no represented normal exit from the repeating path.

## Side effects

Observable effects include mixed-width and overlapping memory accesses, global reads and assignments, indirect invocations, a possible breakpoint, and potentially unbounded execution.

Global storage may overlap other symbols or computed addresses. No disjointness is assumed. Dependencies may change globals and referenced memory before later observations.

No rollback, cleanup, error recovery, or explicit error reporting is represented. Faults, nonreturning dependencies, or breakpoint delivery may stop progress after earlier effects.

The stated sequence is the represented local order. It does not establish volatile semantics, inter-thread synchronization, physical transaction sizes, atomicity, alignment tolerance, or device-bus behavior.

## Dependencies

Opaque IDs identify represented call sites, not verified external identities.

| ID | Target and interface |
|---|---|
| F-opaque-01 | Link value with its low two bits cleared, plus `0x8a4`; three arguments; associated secondary result contributes to a later address. |
| F-opaque-02 | Terminal target base plus `0x0897f314`; four arguments; represented return permits function completion. |
| F-opaque-03 | Dynamic target base plus `0xa1e`; one argument; 64-bit result. |
| F-opaque-04 | `0x0897edcd` plus `0x3c4`; no explicit arguments. |
| F-opaque-05 | `0x0897eddf` plus `0xffe`; no explicit arguments. |
| F-opaque-06 | `0x0897ede1` plus `0x800`; one 32-bit argument. |
| F-opaque-07 | Fixed represented target `0x07f755f6`; one 32-bit argument; 64-bit result. |
| F-opaque-08 | `0x0897edf1` plus `0xae`; one cursor-derived argument. |
| F-opaque-09 | Represented code-address anchor plus `0xffe`; no explicit arguments; 64-bit result. |

F-opaque-03 initially uses base `0x08bfed9a`. Short restarts use `0x0897edf1`; header continuations use `0x0897ee1d`; resumed breakpoint continuations use `0x08790d52`.

No dependency is assumed to preserve globals or pointed-to memory, own or release storage, accept invalid addresses, or return normally. Absence of explicit arguments does not establish absence of hidden inputs or side effects.

## Acceptance examples

These are conditional static criteria, not executed tests. Each assumes preceding effects complete and any necessary arithmetic interpretation is supplied.

1. **Setup bypass:** D zero excludes setup-specific writes and F-opaque-01, but not the entry global capture or common stores.

2. **Distinct observations:** If the first address-bearing global changes after entry capture, setup’s `0xce` byte uses the later observation. Cursor derivation uses the entry capture.

3. **Completed setup:** The common halfword targets `0x30`; the common byte and data-word writes target zero. Repeating-path execution is selected.

4. **Ordered overlap:** With setup bypassed, the zero byte at `0xa1` precedes the data-word store there. Matching final ordinary-memory contents alone is insufficient.

5. **Terminal completion:** If the terminal predicate holds and F-opaque-02 returns normally, no repeating-path initialization follows.

6. **Short restart:** A scaled value below `0x14` causes the next F-opaque-03 invocation to receive zero and use base `0x0897edf1`, without another header visit.

7. **Restart after breakpoint continuation:** Even if the preceding F-opaque-03 invocation used the header-derived argument, a subsequent below-threshold result resets the next argument to zero.

8. **Conditional zero-address store:** Under wrapping scale-by-32 semantics, a below-threshold result requires a word-zero store at zero before the short restart.

9. **Cursor lifetime:** The old cursor determines the step-read address and cursor-plus-four payload before the high-half-derived read replaces it.

10. **Extended overlap:** The twice-L byte store and L-plus-eight byte store precede the 64-bit store at L, even if later writes overwrite their effects.

11. **Continuation uncertainty:** Neither header re-entry nor breakpoint execution may be selected solely by choosing a convenient interpretation of the width-sized shift.

## Uncertainties

The following remain **documented_only**:

- Completeness of the function boundary and hidden-state interface.
- Missing global declarations, physical access widths, and alias relationships.
- Memory mappings, permissions, endianness, and alignment behavior.
- Signed overflow, signed shifts, pointer arithmetic, and the right shift by 32.
- Indirect-transfer conventions, hidden effects, target validity, and return behavior.
- Physical transaction sizes, concurrency properties, and atomicity.
- Breakpoint delivery, resumption, and external intervention.
- Application intent and meaningful termination conditions.

These unknowns do not authorize omitted accesses, invented validation, assumed ownership, or a predetermined continuation outcome.

## Evidence

Established behavior derives from the supplied static representation’s explicit access widths, constants, value dependencies, invocation interfaces, and control-flow relationships.

The short restart’s zero argument follows from leaving the inner cycle and re-entering the outer iteration, where that argument is initialized again. The target base remains `0x0897edf1` on that route.

Hardware-oriented intent is inferred, not established. Storage labels and embedded comments do not supply missing declarations or prove runtime semantics.

No runtime tests, emulation, compilation, external inspection, or implementation execution have been performed.
