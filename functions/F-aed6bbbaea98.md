<!-- Trusted provenance: F-aed6bbbaea98; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-aed6bbbaea98

## Purpose

The supplied static reconstruction describes a state-dependent routine with indirect calls, computed-address memory accesses, a coprocessor operation, and potentially unbounded repetition. Two paths construct a 64-bit return value. Another path ends at an invalid-instruction boundary after a memory write.

No application-level purpose is established. The evidence does not identify a parser, allocator, renderer, or conventional object operation.

This contract describes observable behavior represented by the reconstruction. It does not establish that every represented path is executable on the original machine. Execution depends on implicit entry state, accessible memory, and unresolved dependency contracts.

## Interface

### Explicit inputs

The visible interface has four 32-bit inputs, designated A through D:

| Input | Visible use |
|---|---|
| A | Base for a 16-bit write at displacement 60 on the invalid-instruction path. |
| B | Displacement added to C for a byte read on that path. |
| C | Base for that byte read. |
| D | Source for a subtract-and-shift argument supplied to F-U03. |

The reconstruction represents these inputs as signed integers. Address uses consume their bit patterns; no pointer validation is visible.

### Implicit inputs

The effective interface also includes:

- An entry condition flag selecting the initial branch.
- An entry return-link value used to form the first indirect-call target.
- An entry pointer whose byte at displacement 1 becomes a destination address.
- Three additional entry register values: stored data, a subtraction operand, and an address displacement.
- Coprocessor operand state.
- Global storage read at several distinct execution points.
- Supplementary register results associated with calls whose apparent return types do not fully describe those results.
- Stack storage exposed to a dependency.

Four explicit arguments alone do not determine execution.

### Output

On either normal-return path, the result is a 64-bit bit pattern comprising:

- **Upper 32 bits:** the return-time global word, designated GR, minus 84.
- **Lower 32 bits:** a path-selected 32-bit word shifted left once.

The lower word comes either from the low word returned by F-U05 or from an arithmetic right shift of supplementary register state observed after F-U13.

The evidence does not establish how callers interpret the 64-bit result.

Normal return is not guaranteed. Execution may encounter the invalid-instruction boundary, fault during an access or coprocessor operation, fail or fail to return inside a dependency, or repeat indefinitely.

### Widths and interpretation

Memory accesses have explicit widths of 8, 16, or 32 bits. Pair-valued results contain two 32-bit words.

The behavioral arithmetic below uses 32-bit bit patterns, retaining the low 32 bits of additions, subtractions, multiplications, and left shifts. Halfword and byte stores retain the low 16 or 8 bits respectively. Address arithmetic follows that same 32-bit interpretation.

Signed right shifts are described as arithmetic shifts, consistent with the reconstruction. These interpretations are not guarantees of portable signed-language arithmetic or independently verified machine instructions.

Byte order, alignment restrictions, valid address ranges, and fault delivery remain unresolved. Successful completion of each access requires the corresponding environment-specific memory preconditions.

## Behavior

### Entry selection

The first visible operation reads a 32-bit global word, designated G1.

If the implicit entry condition flag is clear:

1. Read a byte at C + B.
2. Convert it through the reconstructed character type and store the resulting 16-bit value at A + 60.
3. Reach the invalid-instruction boundary.

No normal return or indirect call is represented on this branch. The G1 read precedes the byte read, and the byte read precedes the halfword write.

For bytes below 128, the stored value is unambiguous. For bytes with the high bit set, the reconstruction does not independently settle whether the intermediate character conversion sign-extends or zero-extends.

If the entry flag is set, execution proceeds as follows.

### Initial calls and writes

Call F-U01 with no explicit arguments. Its target is formed from the entry return-link value plus 4,026.

After it returns:

1. Read an unsigned byte from the implicit entry pointer plus 1. Designate the resulting numerical address P.
2. Read a 32-bit word at the F-U01 result plus 100. Designate this word Q.
3. Issue a coprocessor operation using Q and implicit coprocessor state.
4. Read a 32-bit word at Q + 20.
5. Call F-U02 with the F-U01 result and that newly read word. Designate its 32-bit result R.

Then perform these stores in the listed order:

| Address | Width | Stored value |
|---|---:|---|
| P + Q | 8 bits | Low byte of the entry snapshot G1 |
| P | 32 bits | R |
| P + 4 | 32 bits | P |
| P + 8 | 32 bits | Implicit entry stored-data value |
| P + 12 | 32 bits | Entry snapshot G1 |

P ranges from 0 through 255. It is used directly as an address, not as an index into an inferred allocation. The four word stores therefore require low-address memory access.

Addresses may alias each other or other storage used by the routine. The listed order applies even when a later store overwrites an earlier store. Calls may alter global storage, but these stores use the G1 value already read at entry.

### Pair-producing call and initial routing

Before calling F-U03, record whether the unsigned interpretation of R shifted arithmetically right by 9 is greater than `0xFFFFFFBE`.

Call F-U03 with three 32-bit arguments:

- R.
- R shifted arithmetically right by 9, then increased by 65.
- D decreased by 60, then shifted arithmetically right by 20.

F-U03 supplies a 64-bit pair. Designate its low word L and high word H.

Under the stated shift interpretation, the recorded predicate is true exactly for signed R in the inclusive range −33,280 through −1. The predicate is recorded before the call; the reconstruction does not recompute it afterward.

If the predicate is false, enter the shared continuation.

If it is true:

1. Store the low 16 bits of Q at L + 42.
2. Compare Q’s sign bit with the signed-overflow indicator for the 32-bit subtraction H − 47.
3. If those Boolean values are equal, enter the shared continuation.
4. Otherwise enter the loop body with:
   - Working difference: the implicit entry subtraction operand minus Q.
   - Working unsigned argument: L shifted logically right by 25.
   - Current high word: H.
   - Initial dispatch selector.

The comparison is not an ordinary signed comparison between H and 47. It combines Q’s sign with overflow from a separate subtraction.

### Route-dependent dispatch

F-U04 designates a **call-site family**, not one established callee identity. Its target depends on how the loop body is reached.

| Route into the F-U04 call | Dispatch selector | Target construction | Resulting target value |
|---|---:|---|---:|
| Directly from the predicate-true initial route, bypassing the shared continuation | `0x08573355` | Clear the low two bits, then add `0x4A8` | `0x085737FC` |
| After the shared continuation reads a nonzero condition and repeats | `0x08573459` | Clear the low two bits, then add `0x4A8` | `0x08573900` |

The second selector is installed before F-U13. A subsequent nonzero condition takes execution to the loop body with that selector in the reconstruction. This applies both when the shared continuation was reached directly from F-U03 and when it followed a completed longer loop-body path.

Thus the first F-U04 call need not use the initial target: if execution first passes through the shared continuation, its first F-U04 call uses the continuation-derived target. The two reconstructed target values are distinct; their runtime identities and behavior remain unknown.

F-U11 has a separate route distinction. Its selector is the initial selector when reached directly from F-U03, but a different selector, `0x08573405`, is installed after F-U10 on the longer loop-body path. F-U11 adds `0xAAA` without the F-U04 low-bit mask. These are distinct target constructions, not interchangeable uses of one inferred dependency.

### Loop body

Each entry into the loop body performs:

1. Call the route-selected F-U04 target with the working unsigned argument, current high word, and an address into stack storage.
2. Call F-U05 with no explicit arguments, receiving a low/high 32-bit pair.
3. Compute T as the current high word plus 10.
4. Store the working difference as a 32-bit word at the current high word plus 62.
5. Inspect the sign bit of the 32-bit product T × 65,536.

If that product has its sign bit set, return immediately:

- Lower result word: the F-U05 low word shifted left once.
- Upper result word: a fresh read of GR minus 84.

The store at the current high word plus 62 precedes this return decision. Under the 32-bit interpretation, the decision tests bit 15 of T.

Otherwise:

1. Store the low 16 bits of the F-U05 low word at the F-U05 high word plus 60.
2. Call F-U06 with no explicit arguments.
3. Call F-U07 with no explicit arguments.
4. Store the 16-bit value 38 at the F-U07 result plus 2.
5. Compute V by multiplying T by `0x10000000`, retaining 32 bits, then shifting arithmetically right by 9.
6. Compute W as V + T.
7. Read global word G2.
8. Call F-U08 with the F-U07 result shifted left once and T × 65,536 − 81.
9. Replace W with W XOR the previously read G2.
10. Compute J as V + 47.
11. Store W in stack-relative storage at displacement 304.
12. Read a byte at W + 1.
13. Call F-U09 with a fixed address-like value, that byte, J, and supplementary result-register data.
14. Store J as a 32-bit word at W + 36.
15. Call F-U10 with no explicit arguments.

The pair passed onward is then:

- Low word: W.
- High word: supplementary result-register state observed after F-U10, plus 48.

The F-U11 dispatch selector is replaced with the longer-body selector described above, and execution enters the shared continuation.

The stack-relative word at displacement 304 is distinct from the stack address passed to F-U04 in the reconstruction. Neither region’s usable bounds are established.

### Shared continuation

For the current pair L and H, compute:

- Working unsigned argument: L shifted left once.
- Working difference: 4 × H², retaining 32 bits.

The reconstruction expresses the latter as chained multiplication by H and 2. The equivalent low-32-bit value does not imply that the intermediate signed products are portable language operations.

Then:

1. Call the route-selected F-U11 target with L plus the implicit entry displacement.
2. Read global word G3.
3. Call F-U12 with G3.
4. Install the continuation dispatch selector used by a subsequent F-U04 call.
5. Call F-U13 with no explicit arguments.
6. Read global condition G4.

If G4 is zero, return:

- Lower word: supplementary result-register state observed after F-U13, shifted arithmetically right by 9 and then left once.
- Upper word: a fresh read of GR minus 84.

If G4 is nonzero:

1. Store the working difference as a 32-bit word at 2 × H + 52.
2. Replace the current high word with the supplementary result-register value observed after F-U13.
3. Enter the loop body using the continuation-derived F-U04 target.

The working unsigned argument remains the doubled low word computed before the continuation calls. The store address uses the earlier pair’s H, while the next loop-body current high word comes from the supplementary result.

No iteration bound, progress condition, cancellation mechanism, or cleanup path is visible.

## Side effects

The routine directly reads and writes byte, halfword, and word storage through explicit inputs, low numerical addresses, dependency results, globals, implicit register values, and stack-relative addresses.

Some addresses are not inherently aligned for their access widths. Whether such accesses succeed is environment-dependent. No atomicity or synchronization guarantee is established.

The coprocessor operation occurs before F-U02. Its complete operand interpretation, device effects, privilege requirements, and possible faults are unknown.

No allocation, deallocation, reference counting, or ownership transfer is established. Storing P at P + 4 does not establish an object layout or ownership relationship. Receiving an address-like result does not establish a right to release its storage.

F-U04 receives a stack address. Its read/write extent and whether it retains that address are unknown; retention beyond the storage lifetime cannot be assumed safe.

No rollback or status-code error handling is visible. A fault or nonreturning dependency may leave preceding writes observable. Dependencies may have additional effects, including mutation of global storage. No preservation or memory-management contract is assumed for them.

## Dependencies

The following opaque IDs identify visible call sites or call-site families, not proven unique runtime functions.

| ID | Visible interface and dispatch constraints |
|---|---|
| F-U01 | No explicit arguments; 32-bit address-like result. Target depends on the entry return-link value. |
| F-U02 | Two explicit 32-bit arguments; returns R. |
| F-U03 | Three explicit 32-bit arguments; returns a low/high pair. |
| F-U04 | Two 32-bit scalar arguments and a stack address. Initial-route target is `0x085737FC`; post-continuation target is `0x08573900`. These are distinct reconstructed targets. Primary return unused. |
| F-U05 | No explicit arguments; returns a low/high pair. |
| F-U06 | No explicit arguments; primary return unused. |
| F-U07 | No explicit arguments; returns a write base. |
| F-U08 | Two explicit 32-bit arguments; primary return unused. |
| F-U09 | Four visible arguments, including a fixed address-like value, a byte-derived value, and supplementary register data. |
| F-U10 | No explicit arguments; subsequent pair construction consumes supplementary register state. |
| F-U11 | One explicit 32-bit argument; distinct target calculations for direct initial-continuation entry and entry after the longer loop body. |
| F-U12 | One global-derived 32-bit argument. |
| F-U13 | No explicit arguments; supplementary register state is subsequently consumed. The continuation dispatch selector is installed before this call. |

F-882dff43a182 maps to the shared return location in the supplied material. No ordinary call to it or independently callable contract is established.

## Acceptance examples

These are conditional static examples, not executed tests. Each assumes successful preceding accesses, the stated arithmetic interpretation, and dependencies that return as required.

1. **Clear entry flag.** If the byte at C + B is 127, execution reads G1, reads that byte, stores the 16-bit value 127 at A + 60, and reaches the invalid-instruction boundary. F-U01 is not reached.

2. **Low-address destination.** If P is 128, the four initial word stores target 128, 132, 136, and 140. The byte store at P + Q precedes all four; no separate allocation base is introduced.

3. **Predicate boundary.** R = −33,280 selects the predicate-true route and its halfword store at L + 42. R = −33,281 and R = 0 select the shared continuation directly.

4. **Initial-route F-U04 target.** Suppose the predicate is true, Q is negative, and H − 47 does not overflow. After the halfword store, execution enters the loop body directly. Its F-U04 target is `0x085737FC`, not the continuation-derived target.

5. **First call after a continuation.** Suppose the predicate is false and G4 is nonzero after F-U13. The first F-U04 call uses `0x08573900`, even though no previous loop-body iteration occurred. The continuation store at 2 × H + 52 precedes that call.

6. **Target transition across iterations.** Suppose execution first reaches F-U04 through the initial route, completes the longer body, and then reads nonzero G4 in the shared continuation. The first F-U04 target is `0x085737FC`; the next is `0x08573900`. The latter selector was installed before F-U13, not inherited unchanged from the initial route.

7. **Loop-body early return.** If T has bit 15 set, the word store at the current high word plus 62 still occurs before returning. The later halfword store through the F-U05 high word and calls F-U06 onward are not reached on that iteration.

8. **Zero continuation condition.** If G4 is zero when read after F-U13, the continuation store and subsequent F-U04 call do not occur. The lower return word uses the supplementary post-F-U13 value; the upper word uses GR read at return time.

9. **Dependency mutation of globals.** If F-U13 changes G4 before returning, the following read observes the resulting storage value. If a dependency changes GR before normal return, the upper return word is not constrained to its entry value.

## Uncertainties

The following remain **documented_only**:

- Application purpose and validity of every reconstructed control-flow edge.
- Exact instruction-set state, calling convention, and supplementary register-result provenance.
- Character signedness on the invalid-instruction path.
- Machine confirmation of wrapped arithmetic and arithmetic shifts.
- Endianness, alignment behavior, address validity, and fault handling.
- Complete coprocessor semantics.
- Runtime identities and contracts of all indirect targets, including both distinct F-U04 targets.
- G4’s access width and semantic meaning.
- Stack storage bounds and dependency retention of exposed addresses.
- Whether any actual execution satisfies all prerequisites for normal return.
- Whether the mapped shared return location has a separate callable meaning.

The visible distinction between F-U04 targets is established in the reconstruction; it is not an unresolved dependency-identity question. Unknown identities do not justify treating the two targets as one.

No guarantee of termination, determinism, reentrancy, thread safety, ownership transfer, or dependency preservation follows from this evidence.

## Evidence

**Established static evidence:** the supplied reconstruction contains four explicit 32-bit inputs; an implicit entry-flag branch; ordered accesses and calls; two distinct masked-and-offset F-U04 target calculations selected by route; a separate route-dependent F-U11 calculation; a coprocessor operation; repeated continuation logic; and two routes to a constructed 64-bit return.

**Inferred interpretation:** modular 32-bit arithmetic, arithmetic signed shifts, and low/high result pairing provide a coherent behavioral account. Numeric dispatch targets above are derived from the displayed selector, mask, and offset operations, not runtime observations.

**Limitations:** the supplied material warns of invalid instruction data and overlapping global symbols. Accordingly, global storage designations need not be disjoint, and represented call signatures or control-flow edges may be incomplete.

No runtime tests or independent machine-level verification have been performed.
