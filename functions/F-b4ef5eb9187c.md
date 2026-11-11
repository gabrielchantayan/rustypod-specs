<!-- Trusted provenance: F-b4ef5eb9187c; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-b4ef5eb9187c

## Purpose

F-b4ef5eb9187c repeatedly performs an ordered sequence of indirect calls, memory accesses, address calculations, and a software breakpoint operation.

The supplied static representation has unconditional repetition and no normal return path. An iteration reaches its successor only if every intervening call returns, every memory operation completes, and execution continues after the breakpoint.

No application-level purpose is established. Interpretations such as initialization, device control, or event processing remain unsupported.

## Interface

### Inputs and represented state

The represented interface has no explicit parameters and no explicit return value. Its behavior nevertheless depends on incoming machine state, external locations, and dependency results.

The following contract labels describe uses rather than recovered application types:

| Label | Meaning |
|---|---|
| **L** | Incoming link-derived value used to select the first call target. Replaced near the end of each completed iteration. |
| **B** | Register-like address base used for several word accesses. Explicitly decreased by `0xF4` during each iteration that reaches that update. |
| **R** | Separate incoming register-like value contributing to a late byte-store address. |
| **G1**, **G2** | Distinct external locations read as 32-bit values at different points in each iteration. |

B is not established to be a valid stack pointer. G1 and G2 are not established to contain owned objects.

Additional inputs include memory contents, three represented 64-bit dependency results, primary dependency results of narrower widths, and auxiliary register-derived results. Calls without explicit arguments may still consume machine state.

### Widths and arithmetic

The represented accesses use 8-bit bytes, 16-bit halfwords, and 32-bit words. Address calculations use represented 32-bit values.

F-05, F-09, and F-28 each supply a represented 64-bit result. This contract separates each result numerically into its low and high 32-bit components. That separation neither establishes memory byte order nor proves that the actual interface returns one logical 64-bit object rather than grouped register outputs.

Unsigned halfword reads produce values from 0 through 65,535. Byte and halfword stores retain the low 8 or 16 bits of their supplied values.

The right shifts described below are logical. The left shift of F-09’s low component is also explicitly unsigned: shifting that component left by two produces its low-32-bit result. Other calculations involve represented signed operands, including some address additions, left shifts, and multiplication by four. Their overflow behavior is not established as a portable source-language contract. Examples avoid those boundaries unless explicitly conditional.

### Outputs and preconditions

There is no explicit result, success indicator, or error status. Observable behavior consists of dependency invocations, memory accesses, the software breakpoint, and represented state updates.

Successful progress requires each addressed location to support the specified access when that access occurs. Initial validity is insufficient if an intervening dependency changes memory or lifetime. No alignment, extent, nullability, ownership, or preservation guarantees are supplied.

## Behavior

Each phase below is conditional on completion of all earlier phases. F-01 through F-37 are opaque call-site identifiers. Equal-target relationships are listed under Dependencies; they do not merge invocations.

### Opening calls and byte effects

A retained byte is initialized to zero before F-01. F-01 targets L plus `0xFFE` and receives two zero arguments. A second byte is initialized to zero after F-01, before F-02 and F-03 execute in that order.

After F-03, G1 is read as a word and retained as address base **A**. F-04 then executes. F-05 supplies a represented 64-bit result with low component **p** and high component **q**.

F-06 receives p as its first argument and the represented sum of q, p, and the second zero byte as its second argument. Following F-06, an auxiliary byte result is stored at A + `0x11`. This byte is not established as F-06’s primary return value.

F-07 executes next. The first retained zero byte is then stored at A + `0x15`. F-08 follows and supplies a 16-bit primary result, which is stored at an auxiliary address-bearing result plus `0x0C`.

Both A-relative stores use the value captured from G1 before F-04. No reload of G1 is shown between them.

### First paired result and pre-breakpoint accesses

F-09 supplies a represented 64-bit result whose low and high components are **x** and **y**. A zero halfword is stored at y + `0x2C`. Next, x logically shifted right by 26 bits is stored as a word at B + `0x26C`; this stored value is between 0 and 63.

The constant **K** is established as `0x21C48960`. After F-10, a word is read from B + `0x320` and retained as address base **C**. F-11 and F-12 execute, followed by a word store of x shifted left by two at C + `0x54`. This shift uses unsigned 32-bit arithmetic, retaining the low 32 bits.

F-13, F-14, F-15, F-16, and F-17 execute in order. An unsigned halfword is then read from K + `0x34`, namely address `0x21C48994`, and retained as **h**. F-18 follows.

A word is read from B + `0x280` and supplied to F-19. Its represented 32-bit result is retained. The software breakpoint with immediate `0x92` occurs immediately afterward.

At this boundary, **six direct stores** precede the breakpoint: two byte stores, two halfword stores, and two word stores. In particular, both the B + `0x26C` store and the C + `0x54` store occur before F-19 and the breakpoint.

### Resumption and intermediate halfword effects

If execution resumes after the breakpoint, F-19’s retained result is shifted left by two to form **j**.

A second quantity, **t**, is derived by logically shifting K right by 18 bits and multiplying by four. Its established value is `0x21C4`.

A word is read from h + `0x44` and retained as address base **D**. F-20 then executes, followed by F-21, whose word result is retained as **w**. A word is read from h + `0x04`; F-22 receives w and this loaded word.

Both h-relative reads use the zero-extended halfword value h itself as their address base. No recovery of high address bits or additional pointer lookup is shown.

G2 is then read as a word and retained as **E**. F-23 executes before a halfword store of E’s low 16 bits at E + j. F-24 follows. The destination and stored value both derive from the captured E; a change to G2 during F-23 does not itself replace E.

A word is next read from t + `0x54`, which is address `0x2218`, and retained as address base **H**. F-25 supplies address-bearing primary result **P**. The halfword `0x21C4` is stored at P + `0x04`.

F-26 and F-27 execute in order. An unsigned halfword is then read from H + `0x34` and retained as **u**. F-28 follows and supplies the third represented 64-bit result, with low component **a** and high component **b**.

### Base movement and final effects

The word a is stored at B + `0x258`, using B before its explicit decrease. An unsigned halfword is then read from b + `0x14` and retained as **v**.

The represented calculations form **s** by shifting a left six bits and **z** by multiplying b by four. B is then decreased by `0xF4`. The quantity **r** is formed as four times u, and a is stored as a word at r + `0x74`.

F-29 executes before a word store of v at D + `0x54`. F-30 then executes before another word store of the same retained v at D + `0x64`. Both stores write a zero-extended value between 0 and 65,535; there is no intervening reload from b + `0x14`.

F-31 executes and supplies auxiliary address-bearing result **Q**. The low 16 bits of D are stored at s + Q.

F-32 and F-33 execute in order. One byte is then read from u + r and retained as **c**. F-34 executes before c is stored at v + R + `0x19`. The byte is retained across F-34 rather than explicitly reread afterward.

F-35 executes before z is stored as a word at D + `0x44`. F-36 follows.

L is replaced with `0x08712031`. A word is read from z + `0x44` and supplied to F-37. If F-37 returns and execution continues, the next iteration begins unconditionally.

The final read is through z, not D. The earlier store at D + `0x44` determines that read’s value only under suitable aliasing and absent intervening changes.

The represented next iteration carries forward the decreased B and replacement L. With that replacement L, the next F-01 target expression evaluates to `0x0871302F`. The original incoming L is not explicitly restored.

## Side effects

### Direct memory writes

A complete iteration contains **15 explicit stores: three byte stores, five halfword stores, and seven word stores**.

| Width | Destinations, in execution order within that width |
|---|---|
| Byte | A + `0x11`; A + `0x15`; v + R + `0x19` |
| Halfword | F-08 auxiliary base + `0x0C`; y + `0x2C`; E + j; P + `0x04`; s + Q |
| Word | B + `0x26C`; C + `0x54`; pre-decrease B + `0x258`; r + `0x74`; D + `0x54`; D + `0x64`; D + `0x44` |

The Behavior section establishes their interleaving with calls and reads. These counts exclude dependency effects and any breakpoint-handler effects.

No non-aliasing relationship is established. A destination may overlap an earlier destination, a later source, an external location, or dependency-visible memory. Consequently, the behavior cannot be reduced to an unordered collection of final assignments.

### State, ownership, and interruption

The representation explicitly decreases B and replaces L. It does not explicitly update R.

No allocation, release, reference-count operation, or ownership transfer is shown. Address-bearing results and truncated address values do not establish ownership. Unknown dependencies must not be assumed to preserve globals, preserve addressed objects, or acquire or release memory.

No explicit null, alignment, bounds, or lifetime checks occur. No explicit rollback, recovery branch, or error return is shown.

A nonreturning call, faulting access, or breakpoint that does not resume prevents later effects. Earlier completed writes are not shown being undone. Access atomicity, partial-access visibility, exception handling, and breakpoint disposition remain unresolved.

## Dependencies

The opaque identifiers distinguish invocation sites, including sites that select the same target.

| Call sites | Explicitly represented interaction |
|---|---|
| F-01 | L-derived target; receives two zero arguments. |
| F-02–F-04 | No explicit arguments or consumed primary results. |
| F-05 | Supplies the first represented 64-bit result. |
| F-06 | Receives two computed word arguments; supplies an auxiliary byte subsequently stored. |
| F-07 | Separates the first two byte stores. |
| F-08 | Supplies a halfword primary result and auxiliary destination base. |
| F-09 | Supplies the second represented 64-bit result. |
| F-10–F-18 | Interleave the B-relative, C-relative, and K-relative accesses. |
| F-19 | Receives a B-relative word; supplies the result scaled into j after the breakpoint. |
| F-20–F-22 | F-21 supplies w; F-22 receives w and the word read through h. |
| F-23–F-24 | Surround the E-derived halfword store. |
| F-25 | Supplies P. |
| F-26–F-27 | Precede the unsigned halfword read through H. |
| F-28 | Supplies the third represented 64-bit result. |
| F-29–F-30 | Precede the respective word stores of v through D. |
| F-31 | Supplies auxiliary address-bearing result Q. |
| F-32–F-36 | Interleave the final byte read, byte store, and word store. |
| F-37 | Receives the word read through z and precedes repetition. |

Evaluating the supplied fixed target expressions establishes these equal-target pairs:

- F-05 and F-06.
- F-09 and F-13.
- F-17 and F-24.
- F-19 and F-31.
- F-25 and F-32.

Target equality does not establish interchangeable behavior. Each invocation occurs at its own point with potentially different arguments, machine state, and consumed outputs. Differing result uses also do not establish a complete shared signature.

Except for F-01’s dependence on L, target expressions use fixed internally selected values and offsets. Dependency identities, complete calling conventions, preservation rules, lifetime effects, and semantic contracts remain unknown.

## Acceptance examples

These examples are conditional consequences of the static representation, not executed tests. Each assumes that preceding operations complete and relevant accesses are valid.

1. **Unsigned extraction and shift.** If F-09 supplies x = `0xFC000003`, the word at B + `0x26C` receives 63. The later word at C + `0x54` receives `0xF000000C`, the unsigned 32-bit result of shifting x left by two.

2. **Zero-extended address base.** If h is `0xFFFF`, the two h-relative word reads occur at `0x10043` and `0x10003`. No sign extension to a negative base is represented.

3. **Captured external value.** If E is `0x12345678` and F-19 returns 3, j is 12. The E-derived halfword store targets `0x12345684` and writes `0x5678`. A change to G2 during F-23 does not by itself change this retained destination or value.

4. **Fixed low-address access.** The word read producing H uses address `0x2218`. If F-25 returns P = `0x1000`, the following halfword store writes `0x21C4` at `0x1004`.

5. **Value retained across a call.** If v is 500, the stores at D + `0x54` and D + `0x64` both write the word value 500, despite F-30 occurring between them. This does not assert that either destination remains unchanged afterward.

6. **Maximum u.** If u is 65,535, r is 262,140. The word-store address r + `0x74` is 262,256, and the byte-read address u + r is 327,675. These calculations do not overflow 32 bits.

7. **Breakpoint stops continuation.** If execution reaches the breakpoint but does not resume, six direct stores precede it: the bytes at A + `0x11` and A + `0x15`; the halfwords at the F-08-derived destination and y + `0x2C`; and the words at B + `0x26C` and C + `0x54`. No post-breakpoint operation occurs. No rollback is shown.

8. **Repeated B movement.** If two iterations reach the explicit B decrease, with no external alteration of the represented state and no arithmetic-boundary issue, the cumulative decrease is `0x1E8`. The second iteration’s early B-relative accesses use the value carried from the first decrease.

9. **Equal targets remain separate invocations.** F-05 and F-06 select the same established target. F-05’s represented result contributes to F-06’s arguments; target equality does not permit either invocation to be omitted.

10. **Different final bases.** If D and z designate different nonoverlapping regions, the store at D + `0x44` does not itself supply the word subsequently read from z + `0x44` for F-37.

## Uncertainties

The following remain **documented_only**:

- Whether the represented function boundary and parameterless interface match the actual calling convention.
- Complete dependency signatures, identities, effects, register-preservation rules, and lifetime guarantees.
- Target relationships beyond the equalities explicitly listed.
- The ABI meaning and reliability of auxiliary register-derived results.
- Whether the represented 64-bit results are logical single results or grouped outputs.
- Address validity, alignment, object extent, ownership, and lifetime.
- Whether accessed storage is ordinary, shared, or device memory.
- Byte order, atomicity, volatility, synchronization, and fault behavior.
- Overflow behavior for calculations involving represented signed operands.
- Breakpoint handling and whether execution resumes.
- Whether unconditional repetition and B movement reflect intended execution structure.
- Application-level intent.

The labels “base,” “component,” and “result” describe represented uses. They do not resolve these uncertainties.

## Evidence

The contract rests solely on the supplied static representation. It establishes call order, explicit access widths, interleaving of reads and writes, breakpoint placement, represented state updates, unconditional repetition, three represented 64-bit results, and 15 direct stores.

The pre-breakpoint count follows from the six stores explicitly preceding the breakpoint: two bytes, two halfwords, and two words. Both pre-breakpoint word destinations are included.

The target equalities, t = `0x21C4`, the read address `0x2218`, the next L-derived target, and the bounds on u-derived arithmetic are static arithmetic deductions.

Application intent is not inferred. No runtime execution, dependency inspection, smoke run, or tests have been performed. Acceptance examples describe conditional static consequences, not observed outcomes.
