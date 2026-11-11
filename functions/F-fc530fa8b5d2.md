<!-- Trusted provenance: F-fc530fa8b5d2; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-fc530fa8b5d2

## Purpose

The function repeatedly performs indirect calls and ordered memory accesses using implicit register state, global values, and dependency outputs. The visible body has an unconditional loop back-edge and no normal return path.

Each iteration that reaches the relevant operations decreases an implicit address base and assigns a fixed value to a link-like state component. That assignment does **not** unconditionally establish the next iteration’s first call target: another dependency is called before the loop repeats, and its preservation of that state is unknown.

A software breakpoint interrupts the middle of each iteration that reaches it. All subsequent behavior is conditional on execution continuing.

No application-level purpose, object model, resource lifecycle, or successful completion condition is established. Initialization, dispatch, and device-processing interpretations would be inferred intent rather than contractual facts.

## Interface

### Explicit and implicit inputs

There are no declared explicit parameters, no declared result, and no visible normal return.

Execution depends on:

- **B**: an implicit address-base state component used for several word accesses and explicitly decreased late in the iteration.
- **R**: an incoming register-like value contributing to a late byte-store address.
- **Λ**: a link-like value used to calculate the first indirect-call target.
- **G1** and **G2**: global cells read as 32-bit values at different points in the iteration.
- Memory reached through those values, fixed numeric addresses, dependency results, and computed addresses.
- Auxiliary dependency outputs not represented as ordinary return values.
- The environment’s treatment of indirect calls, memory accesses, and the breakpoint.

The visible body performs no explicit validation of these inputs. Dependencies may nevertheless validate, reject, modify, or otherwise act on them before later operations.

### Widths and arithmetic

A byte is 8 bits, a halfword 16 bits, and a word 32 bits. Three dependencies supply represented 64-bit paired results. For a paired result, **lo** and **hi** denote its low and high 32-bit portions.

Halfword loads used in wider arithmetic are zero-extended. Halfword and byte stores retain the corresponding low bits. A word store of a zero-extended halfword writes a full 32-bit value, not merely two bytes.

Address arithmetic below describes the represented 32-bit computations. Addition, multiplication, and left shifts are expressed by their low 32-bit results. Logical right shifts insert zero bits. This notation does not establish language-level signed-overflow behavior or guarantee architectural execution when an operation or address is invalid.

Byte order, alignment requirements, address mapping, and the architectural meaning of B, R, and Λ are **documented_only** unknowns.

### Captured values and live state

A captured memory value is not implicitly reread after intervening calls. This matters for G1, G2, and several pointer-like loads.

B denotes the state at each use, not a separately captured iteration-wide constant. The body’s explicit decrement is established; preservation of its effective state across unknown dependencies is not. Likewise, no general preservation guarantee is assumed for R or Λ.

## Behavior

The ordering below applies only while all preceding calls and accesses complete and execution proceeds past the breakpoint. A nonreturning dependency, fault, or breakpoint stop can prevent all subsequent effects.

### Early calls and object-relative stores

At the iteration’s beginning, the body establishes a zero byte for an early store. It calls **F-10000001** with two zero arguments, targeting the 32-bit sum Λ + `0xFFE`. A separate byte operand later used in an argument calculation is then established as zero.

Calls **F-10000002** and **F-10000003** occur before the 32-bit read of G1. Denote the captured value by **P**. **F-10000004** follows that read.

**F-10000005** supplies paired result **A**. The next call, **F-10000006**, receives lo(A) followed by hi(A) + lo(A), reduced to 32 bits. The additional byte operand in that sum is zero. An auxiliary byte output associated with F-10000006 is then stored at P + `0x11`.

After **F-10000007**, a zero byte is stored at P + `0x15`. Both byte destinations use captured P even if intervening dependencies change G1.

**F-10000008** supplies a halfword result and an auxiliary address-like output. The halfword is stored at that auxiliary base + `0x0C`.

**F-10000009** supplies paired result **D**. Two stores follow, in this order:

- A zero halfword at hi(D) + `0x2C`.
- A word at B + `0x26C` containing lo(D) logically shifted right by 26.

The second stored value is in the range 0–63.

### Captured pointer, narrow address input, and breakpoint

**F-10000010** precedes a word read at B + `0x320`. Denote that captured word by **Q**. Calls **F-10000011** and **F-10000012** intervene before a word store to Q + `0x54`. Its value is lo(D) shifted left by two, retaining 32 bits. The body does not reread B + `0x320` for this store.

Calls **F-10000013**, **F-10000014**, **F-10000015**, **F-10000016**, and **F-10000017** then occur in that order.

Define fixed numeric value **K = `0x21C48960`**. A halfword is loaded at K + `0x34` and zero-extended to **H**. After **F-10000018**, a word is read at B + `0x280` and supplied as the sole explicit argument to **F-10000019**. Its represented 32-bit result is captured before software breakpoint immediate **`0x92`** executes.

If execution continues, that captured result shifted left by two forms **J**. The fixed calculation of K logically shifted right by 18 and then multiplied by four produces **T = `0x21C4`**.

A word read at H + `0x44` supplies captured address-like value **C**. H is a zero-extended halfword, not a full-width pointer. Its possible numeric addresses for this read range from `0x44` through `0x10043`.

### Global-derived and dependency-derived halfword writes

After C has been captured, **F-10000020** runs, followed by **F-10000021**, which supplies word **E**. A word is then read at H + 4. **F-10000022** receives E as its first explicit argument and that newly read word as its second.

G2 is next read as a word and captured as **V**. After **F-10000023**, a halfword containing the low 16 bits of V is stored at J + V. This stores bits of V itself; it is not a halfword load through V.

After **F-10000024**, a word is read at T + `0x54` and captured as **W**. **F-10000025** supplies address-like word **X**. The body stores halfword `0x21C4` at X + 4.

Calls **F-10000026** and **F-10000027** precede a halfword read at W + `0x34`. The zero-extended result is **N**, necessarily between 0 and 65535. **F-10000028** then supplies paired result **Z**.

### Base transition and late indexed effects

Let **L = lo(Z)** and **U = hi(Z)**.

The body first stores L as a word at B + `0x258`. It then loads a halfword at U + `0x14` and zero-extends it to **M**.

The late address calculations use:

- **S**: L shifted left by six, retaining 32 bits.
- **Y**: U multiplied by four, retaining 32 bits.
- **I**: N multiplied by four.

The explicit B update subtracts `0xF4`, or 244. The preceding store at B + `0x258` uses B before this update. I is between 0 and 262140, so this particular multiplication does not overflow 32 bits.

A word containing L is stored at I + `0x74` before **F-10000029**. After that call, M is stored as a word at C + `0x54`. **F-10000030** intervenes before a second word store of M, at C + `0x64`.

**F-10000031** supplies an auxiliary address component. A halfword containing the low 16 bits of C is stored at S plus that component.

Calls **F-10000032** and **F-10000033** precede a byte read at N + I, equivalently 5 × N. Denote the captured byte by **Bv**. **F-10000034** occurs before Bv is stored at M + R + `0x19`; the source byte is not reread after that call.

After **F-10000035**, Y is stored as a word at C + `0x44`. **F-10000036** follows.

### Loop continuation and link-like state

Near the iteration’s end, the body assigns **Λ = `0x08712031`**. It then reads a word at Y + `0x44` and passes that word as the sole explicit argument to **F-10000037**.

If that call returns and execution continues, control goes back to the beginning without a termination test.

The assigned Λ value plus `0xFFE` equals **`0x0871302F`**. This is the next first-call target **only if the assignment survives F-10000037 and any relevant environmental effects**. The supplied material does not establish that preservation.

## Side effects

### Direct stores

An iteration reaching every direct store performs **15 explicit stores**, in this order:

| Order | Destination | Width in bits | Value |
|---:|---|---:|---|
| 1 | P + `0x11` | 8 | Auxiliary output from F-10000006 |
| 2 | P + `0x15` | 8 | Zero |
| 3 | Auxiliary base from F-10000008 + `0x0C` | 16 | Result from F-10000008 |
| 4 | hi(D) + `0x2C` | 16 | Zero |
| 5 | B + `0x26C` | 32 | lo(D) logically shifted right by 26 |
| 6 | Q + `0x54` | 32 | lo(D) shifted left by two |
| 7 | J + V | 16 | Low 16 bits of V |
| 8 | X + 4 | 16 | `0x21C4` |
| 9 | B + `0x258`, before the explicit decrement | 32 | L |
| 10 | I + `0x74` | 32 | L |
| 11 | C + `0x54` | 32 | M |
| 12 | C + `0x64` | 32 | M |
| 13 | S + auxiliary component from F-10000031 | 16 | Low 16 bits of C |
| 14 | M + R + `0x19` | 8 | Bv |
| 15 | C + `0x44` | 32 | Y |

Symbolically different destinations need not be disjoint. Earlier stores can affect later loads through aliasing. Dependencies may also read or overwrite these locations.

The body provides no rollback or transaction boundary. This does **not** guarantee that an earlier stored value remains observable after a later failure: aliasing, intervening dependencies, or environmental behavior may have overwritten or otherwise affected it.

### Other effects, ownership, and errors

The body explicitly updates B and Λ and executes a breakpoint. Dependencies may produce additional state changes.

No allocation, release, ownership transfer, lock operation, or cleanup is explicit in the body. This does not establish their absence inside dependencies. Address-like outputs are used for access without an established ownership or lifetime contract.

For execution to proceed, each reached target must be callable and each reached memory access must be supported by the environment. No explicit null check, bounds check, alignment check, error-result handling, recovery branch, or cleanup path is shown. Unknown dependencies may reject inputs or fail before subsequent body operations. Fault and exception consequences are environment-dependent.

## Dependencies

### Call-site interfaces

The following IDs label call sites, not necessarily different underlying functions. “No explicit arguments” does not exclude implicit inputs; an unused ordinary result does not establish a void interface.

| Call-site IDs | Explicitly consumed interaction |
|---|---|
| F-10000001 | Two zero arguments; target derived from Λ |
| F-10000002–F-10000004 | No explicit arguments; ordinary results unused |
| F-10000005 | Supplies paired result A |
| F-10000006 | Receives two words; auxiliary byte output consumed |
| F-10000007 | No explicit arguments |
| F-10000008 | Supplies halfword result and auxiliary destination base |
| F-10000009 | Supplies paired result D |
| F-10000010–F-10000018 | No explicit arguments; ordinary results unused |
| F-10000019 | Receives one word; supplies word used to form J |
| F-10000020 | No explicit arguments |
| F-10000021 | Supplies word E |
| F-10000022 | Receives two words |
| F-10000023–F-10000024 | No explicit arguments |
| F-10000025 | Supplies word X |
| F-10000026–F-10000027 | No explicit arguments |
| F-10000028 | Supplies paired result Z |
| F-10000029–F-10000030 | No explicit arguments |
| F-10000031 | Auxiliary address component consumed |
| F-10000032–F-10000036 | No explicit arguments |
| F-10000037 | Receives one word |

### Established target relationships

Except for F-10000001, the target expressions reduce to fixed numeric values. The following stable opaque target-group IDs record statically established equalities independently of call-site labels:

| Target-group ID | Call sites sharing that target |
|---|---|
| F-20000001 | F-10000005 and F-10000006 |
| F-20000002 | F-10000009 and F-10000013 |
| F-20000003 | F-10000017 and F-10000024 |
| F-20000004 | F-10000019 and F-10000031 |
| F-20000005 | F-10000025 and F-10000032 |

These five target groups are distinct. The other fixed-target sites have mutually distinct numeric targets and do not match these groups. The dynamic target of F-10000001 is not established as distinct from them.

Equal targets do not establish a complete common interface. For example, F-10000005 consumes a paired ordinary result, while F-10000006 passes two words and consumes an auxiliary byte output. Both nevertheless target F-20000001.

Dependency implementations, register preservation, global preservation, memory retention, ownership, release behavior, and exception behavior remain **documented_only** unknowns.

## Acceptance examples

These are conditional arithmetic and ordering examples, not runtime tests. Each assumes all preceding operations complete and the required addresses are usable.

1. **Six-bit extraction and truncating shift.**  
   If lo(D) is `0xFC000001`, store 5 contains 63 and store 6 contains 4 under the represented 32-bit calculation.

2. **Captured global value.**  
   If G1 yields P and F-10000004 subsequently changes G1, stores 1 and 2 still use P + `0x11` and P + `0x15`. This does not imply either location retains its value after later calls.

3. **Breakpoint stopping execution.**  
   If execution stops permanently at breakpoint `0x92`, the first six explicit stores have been reached. The G2-derived halfword store and explicit B decrement have not.

4. **Zero-extended narrow address input.**  
   If the halfword at K + `0x34` is `0xFFFF`, the C-producing read uses numeric address `0x10043`, not a sign-extended negative address.

5. **Value truncation rather than dereference.**  
   If V is `0x12345678` and J is 8, store 7 targets `0x12345680` and writes halfword `0x5678`. It does not read a halfword at V.

6. **Late indexed effects.**  
   If N is 3, L is 5, U is `0x100`, and the halfword at U + `0x14` is 7, then I is 12, S is 320, and Y is `0x400`. Store 10 writes 5 at `0x80`; stores 11 and 12 write the full word 7. The byte source is address 15. The final explicit call argument is read at `0x444`; its contents depend on intervening effects.

7. **Shared target, different interactions.**  
   Calls F-10000019 and F-10000031 target the same fixed target group, F-20000004, even though the former consumes a word result and the latter consumes an auxiliary address component.

8. **Conditional next target.**  
   If F-10000037 returns without changing the assigned Λ state, the next F-10000001 targets `0x0871302F`. Without that preservation condition, the body’s assignment alone does not establish the next actual target.

9. **Base progression.**  
   If dependencies preserve B and iterations continue, each reached explicit decrement reduces it by `0xF4` in the represented 32-bit arithmetic. The body shows no reset or termination check.

## Uncertainties

- **Application intent — documented_only.** No domain purpose is established.
- **Dependency semantics — documented_only.** Target equalities are established; implementation behavior and complete interfaces are not.
- **Auxiliary output conventions — documented_only.** Consumed widths and uses are visible, but production and validity rules are not.
- **State preservation — documented_only.** In particular, F-10000037 is not known to preserve the assigned Λ value.
- **Memory model — documented_only.** Mapping, alignment, byte order, device semantics, and lifetimes are unspecified.
- **Ownership and validation — documented_only.** No body-level ownership operations or explicit validation are shown; dependency behavior remains unresolved.
- **Arithmetic exceptions — documented_only.** The stated low-word calculations do not supply missing architectural or language-level guarantees.
- **Breakpoint continuation — documented_only.** Later operations require resumed execution.
- **Sustained repetition — inferred intent only.** The back-edge is unconditional, but actual repetition depends on calls returning and execution surviving accesses and breakpoints.

## Evidence

This contract uses static inspection of the supplied function representation only. Established evidence includes access widths, constants, captured values, call and memory-access ordering, fixed-target equalities, the 15 explicit stores, the breakpoint, the B decrement, the Λ assignment, and the unconditional loop back-edge.

Application meaning and dependency behavior are not established by those observations. Acceptance examples express conditional consequences rather than observations of execution.

**No runtime tests have been performed.**
