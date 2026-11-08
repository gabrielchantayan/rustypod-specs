<!-- Trusted provenance: F-2efb421ec8e2; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-2efb421ec8e2

## Purpose

F-2efb421ec8e2 repeatedly invokes indirect dependencies, performs arithmetic on 32-bit values, and accesses computed memory locations. It returns a packed 64-bit value when either of two conditions is satisfied:

- A global gate is zero after the first three dependency calls of an iteration.
- On the continuation path, a signed multiplication result is negative.

Otherwise, it updates the explicit-input state and repeats without an established iteration limit.

The supplied representation establishes a low-level state transformation, not an application-level purpose. It does not establish that the function implements allocation, rendering, parsing, or any particular numerical algorithm.

This contract describes static evidence. Machine arithmetic, incomplete calling conventions, memory validity, and dependency behavior remain conditional where the supplied material does not resolve them. No runtime tests have been performed.

## Interface

### Explicit inputs

| Specification label | Width and presentation | Established use |
|---|---|---|
| U | Unsigned 32-bit input | Arithmetic operand; replaced on continuation by a computed value also used as an address. |
| V | Signed 32-bit input | Arithmetic operand and contributor to a write address; replaced after the final dependency call of a continuing iteration. |

Neither input is validated. Successful execution requires every reached address and call target to be usable; the interface supplies no length, bounds, alignment, or object-validity information.

U and V denote the current iteration’s state, not necessarily their entry values.

### Implicit inputs and auxiliary values

The representation also depends on:

- A link-register-like state, used to form the first indirect call target.
- An incoming register-like word added to U, with no visible assignment establishing its value.
- Three auxiliary word-sized values, designated **X**, **Y**, and **Z** here.
- Four global storage locations.
- The current stack-frame base address.
- Memory reachable through computed addresses.

X is referenced after the third dependency call, as an argument to the fourth call, and after the fifth call. Y supplies an argument to the ninth call. Z supplies the next V after the tenth call.

These labels identify uses visible in the representation. Their producers, register lifetimes, and preservation across intervening dependencies are **documented_only**. They must not be treated as ordinary return values with verified signatures, or assumed unchanged merely because no explicit assignment is shown.

### Global storage

| Label | Address | Visible access |
|---|---|---|
| G-input | `0x08573610` | Word read after the first call and before the second call. |
| G-gate | `0x08573598` | Read and compared with zero after the third call; exact access width unresolved. |
| G-mask | `0x085735CC` | 32-bit read on the full continuation path, before the eighth call. |
| G-result | `0x085735C0` | 32-bit read during return construction. |

The supplied overlap warning prevents treating symbolic globals as independently owned, nonoverlapping objects. Dependencies may modify these locations; no preservation guarantee is established.

### Output

On either normal exit, the represented return is a numerical concatenation of two 32-bit halves:

| Half | Value |
|---|---|
| Upper 32 bits | G-result minus `0x54`, interpreted as a word under wrapping arithmetic. |
| Lower 32 bits | A path-selected word shifted left by one bit. |

The path-selected word is:

- The arithmetic-right-shift-by-nine result of X on the zero-gate exit.
- The low word of the fifth dependency’s represented 64-bit result on the multiplication-controlled exit.

Numerical concatenation does not establish the byte order of any subsequent memory representation.

No explicit error code, status object, or recovery interface is present.

## Behavior

### Arithmetic model

A **word** in this contract is 32 bits. A halfword is 16 bits; a byte is 8 bits.

U’s initial doubling is an unsigned left shift. Other expressions include signed multiplication, signed addition, signed left shift, and right shift of potentially negative signed values. A portable source-language interpretation does not define all possible inputs.

Where this contract gives word-level results, it conditionally uses:

- Truncation to the low 32 bits for overflowing arithmetic.
- Arithmetic right shift for signed right shifts.
- Address arithmetic reduced to the represented 32-bit address space.

These are useful machine-word interpretations, not independently verified target semantics. In particular, interpreting the multiplication-controlled exit as a bit test requires wrapping arithmetic.

### Iteration prefix

The function first derives:

- **D**: U shifted left by one bit.
- **Q**: the represented product V × 2 × V × 2.

Under wrapping arithmetic, Q is the low word of four times V squared. That equivalence does not remove the signed-overflow uncertainty in the supplied expression.

The following observable operations then occur in order:

1. **F-I01** is called at the target formed from the link-register-like state plus `0xAAA`. Its visible arguments are:
   - U plus the incoming register-like word.
   - V.
   - D.
2. G-input is read.
3. **F-I02** is called with the captured G-input word.
4. **F-I03** is called with no visible arguments.
5. G-gate is read and compared with zero.

Consequently, changes to G-input by F-I01 can affect F-I02 in the same iteration. Changes to G-gate by any of the three prefix calls can affect the immediately following branch.

“No visible arguments” does not mean that a dependency lacks implicit register, memory, or global inputs.

### Zero-gate exit

If the gate read is zero, the function leaves the loop without reaching any of the continuation-path direct writes.

The selected lower-half operand is X arithmetic-right-shifted by nine. That result is then shifted left by one for return packing.

G-result is read during return construction and supplies the upper half after subtraction of `0x54`. It is not an entry snapshot.

This exit still follows F-I01, F-I02, and F-I03. Their side effects remain observable.

### Nonzero-gate path through the second write

If the gate read is nonzero, operations proceed as follows:

1. Q is written as a 32-bit value at address 2 × V + `0x34`.
2. **F-I04** is called with D, X, and the current stack-frame base address.
3. **F-I05** is called with no visible arguments and yields a represented 64-bit result.
4. That result is divided numerically into low word **L** and high word **H**.
5. **N** is formed as X + decimal 10.
6. Q is written as a 32-bit value at address X + `0x3E`.
7. The signed result of N × `0x10000` is tested for negativity.

Both 32-bit writes precede the multiplication-controlled exit. The second write follows F-I05; it is not conditional on the result of the subsequent branch test.

The X references describe the supplied representation. Their interpretation as a particular machine register value before and after calls remains unresolved.

### Multiplication-controlled exit

If the tested signed result is negative, the function immediately reaches the shared return construction:

- L supplies the lower-half operand, which is shifted left by one.
- G-result minus `0x54` supplies the upper half.

No halfword write or later dependency call is reached.

Under wrapping 32-bit multiplication, negativity of N × `0x10000` tests bit 15 of N. It does not test whether N itself is negative.

### Full continuation path

If the multiplication-controlled exit is not taken, the following operations occur in order:

1. The low 16 bits of L are written at address H + `0x3C`.
2. **F-I06** is called with no visible arguments.
3. **F-I07** is called with no visible arguments and yields a represented word **P**.
4. The 16-bit value `0x0026` is written at address P + 2.
5. **T** is computed by multiplying N by `0x10000000` as a represented signed word and then right-shifting that result by nine.
6. **A** is formed as T + N.
7. G-mask is read and captured.
8. **F-I08** is called with:
   - P shifted left by one.
   - N × `0x10000` minus `0x51`.
9. **B** is formed by exclusive-ORing A with the previously captured G-mask.
10. **C** is formed as T + `0x2F`.
11. B is written to the represented 32-bit stack-frame slot at offset `0x130`.
12. A byte is read at address B + 1.
13. **F-I09** is called with:
    - The fixed address-valued token `0x085734E8`.
    - The byte just read.
    - C.
    - Y.
14. C is written as a 32-bit value at address B + `0x24`.
15. The represented link-register-like state is assigned `0x08573405`.
16. **F-I10** is called with no visible arguments.
17. V is replaced by Z + `0x30`.
18. Execution repeats, with B as the represented U state.

The link-register-like assignment is specifically **after the write at B + `0x24` and before F-I10**. It is not merely a deferred next-iteration update. Its position must be retained because F-I10’s implicit inputs and register behavior are unresolved.

At the representation level, the next F-I01 target is formed from the updated link-register-like state. This does not independently prove which physical register F-I10 observes, whether it preserves that register, or how the target behaves under a verified ABI.

Likewise, the representation retains the captured mask and computed loop-state values across calls; establishing their concrete machine-level preservation requires information not supplied here.

There is no visible iteration count, progress condition, or termination guarantee.

## Side effects

### Direct writes

On a complete continuation, the function’s represented direct writes occur in this order:

| Order | Width | Destination | Value |
|---:|---:|---|---|
| 1 | 32 bits | 2 × V + `0x34` | Q |
| 2 | 32 bits | X + `0x3E` | Q |
| 3 | 16 bits | H + `0x3C` | Low 16 bits of L |
| 4 | 16 bits | P + 2 | `0x0026` |
| 5 | 32 bits | Stack-frame base + `0x130` | B |
| 6 | 32 bits | B + `0x24` | C |

Calls are interleaved with these writes as specified in Behavior. The table does not authorize moving writes across calls.

The byte read at B + 1 occurs after the stack-slot write and before F-I09. The final 32-bit write occurs after F-I09. Therefore, absent aliasing with earlier writes or dependency mutations, F-I09 can observe the old contents of the final write destination.

No disjointness is established among globals, the stack frame, computed addresses, or dependency-accessible memory. Aliasing may make earlier writes affect later reads or call behavior.

### State-update ordering

The link-register-like assignment is another visible state change, distinct from the six direct writes. It occurs between the final direct write and F-I10.

No conclusion is drawn about whether this assignment is an ABI call setup, a return-address operation, or another register convention. The established requirement is its represented ordering.

### Ownership and lifetime

No allocation, deallocation, or ownership transfer is directly established.

Passing the stack-frame base to F-I04 does not establish its access extent, whether it retains that address, or whether any retained address remains valid after return. The later frame-slot write does not prove an intended communication protocol with that dependency.

Passing `0x085734E8` to F-I09 establishes an argument value, not a pointee layout or ownership convention.

Dependencies may allocate, release, retain, or mutate memory; their contracts are unknown.

### Errors and partial completion

Memory accesses and indirect call targets are unchecked. Successful execution requires all reached reads, writes, and calls to be valid and to complete as needed.

No rollback or fault handler is visible. A fault or non-returning dependency can leave all preceding writes and dependency effects in place. The supplied material establishes no particular exception type, recovery behavior, or return value for such executions.

## Dependencies

These are opaque, specification-local call-site IDs. Different IDs are not proof of different underlying implementations.

| ID | Visible interface | Position |
|---|---|---|
| F-I01 | Three word arguments; no ordinary result consumed | First call of each iteration |
| F-I02 | One captured global-word argument | After G-input read |
| F-I03 | No visible arguments | Before G-gate read |
| F-I04 | Two word arguments and a stack-frame address | Between the first and second direct writes |
| F-I05 | No visible arguments; represented 64-bit result | Before N and the second direct write |
| F-I06 | No visible arguments | After the first halfword write |
| F-I07 | No visible arguments; represented word result | Before the second halfword write |
| F-I08 | Two word arguments | After G-mask capture |
| F-I09 | Fixed address token, byte, word, auxiliary word | After the byte read and before the final direct write |
| F-I10 | No visible arguments | After the final direct write and link-register-like assignment; before V is updated |

Except for F-I01’s state-relative target, targets are formed from embedded constants. Complete ABI signatures, instruction-set-state conventions, and correspondence to independently identified functions are not established.

The supplied mapping also associates **F-882dff43a182** with the shared return location. The visible transfer reaches that location within this body. It does not establish a separate call, an independent interface, or additional side effects.

No dependency is assumed to preserve globals, arbitrary registers, or passed memory.

## Acceptance examples

These examples are conditional static consequences, not executed tests.

### Zero gate

If all three prefix dependencies return and G-gate is zero when read:

- Only F-I01 through F-I03 are reached in that iteration.
- None of the six continuation writes occurs.
- The lower half uses X shifted right by nine, then left by one.
- The upper half uses G-result read during return construction.

If X is `0x00000200` and G-result is `0x00000054`, the packed result is `0x0000000000000002`, subject to the stated arithmetic and auxiliary-value interpretation.

### Exit after two word writes

Assume wrapping arithmetic, a nonzero gate, successful earlier accesses, and X equal to `0x00007FF6`.

N is `0x00008000`; multiplying by `0x10000` produces a word whose sign bit is set. Both Q writes occur, but neither halfword write occurs. F-I06 through F-I10 are skipped. The lower return half is twice L, truncated to a word.

### Negative N without this exit

Under wrapping arithmetic, X equal to `0xFFFFFFF6` produces N equal to zero, so the exit is not taken.

For an actually negative N, `0xFFFF0000` also does not satisfy this condition: its bit 15 is clear, and the wrapped multiplication result is zero. Negativity of N alone is insufficient.

### Captured byte and mask

If the byte at B + 1 is `0x7A` when read, F-I09 receives that captured byte even if it changes the location during its execution.

If F-I08 changes G-mask, the represented B calculation still uses the mask captured before that call. A later iteration may read the changed global.

### Ordered state assignment before F-I10

On a full continuation, an observation of the represented operations must place:

1. The 32-bit write of C at B + `0x24`.
2. The link-register-like assignment of `0x08573405`.
3. Invocation of F-I10.
4. The update of V from Z.

A description placing the assignment after F-I10 is not equivalent. Whether F-I10 observes that assigned state through a physical register is unresolved, not assumed.

### Nontermination and interruption

If every reached dependency returns, every access succeeds, every gate read is nonzero, and every multiplication-controlled exit is false, execution continues indefinitely.

If F-I10 does not return, the final direct write and the preceding link-register-like assignment have already occurred; the subsequent update of V is not reached.

## Uncertainties

The following remain **documented_only**:

- Processor ABI, physical register mapping, and complete dependency signatures.
- Producers and preservation rules for X, Y, Z, and the incoming register-like word.
- F-I10’s relationship to the assigned link-register-like state.
- Exact G-gate access width.
- Target behavior for signed overflow, signed left shifts, and negative right shifts.
- Endianness, alignment requirements, and validity of unaligned accesses.
- Argument extension rules for the byte passed to F-I09.
- Memory-object layouts, bounds, ownership, and lifetime.
- Stack-frame extent and dependency access through its base address.
- Aliasing beyond the supplied overlap warning.
- Dependency side effects, termination, and fault behavior.
- Application-level meaning of the loop and constants.

These uncertainties prevent a fully portable executable contract for every possible input.

## Evidence

Established static evidence is the supplied function representation and opaque dependency mapping. It supports the branch structure, represented call arguments and results, direct access widths other than the unresolved gate width, arithmetic expressions, write/read ordering, loop-state updates, and packed return construction.

In particular, the supplied ordering places the final word write before the link-register-like assignment, and that assignment before F-I10. No ABI interpretation is required to preserve this ordering.

Wrapping arithmetic and arithmetic right shifts are conditional interpretations. Application intent, dependency ownership, physical register behavior, and successful execution on a target are not established.

No runtime tests, traces, or independent machine-level validation have been performed.
