<!-- Trusted provenance: F-97cface4b3d4; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-97cface4b3d4

## Purpose

F-97cface4b3d4 performs an ordered sequence of indirect dependency invocations, four-byte memory transfers, and byte transfers whose destinations are selected by other bytes. If execution reaches its final region, the represented control flow repeats indefinitely.

The supplied static representation does not establish an application-level purpose. In particular, it does not establish allocation, serialization, initialization, device control, or ownership management.

This contract describes the represented accesses, data dependencies, and control flow. Diagnostics accompanying the representation report invalid instruction data, removed unreachable regions, and overlapping global symbols. Correspondence to executable behavior therefore remains qualified.

## Interface

### Explicit and implicit inputs

Three explicit argument positions are represented:

| Position | Represented width | Meaning within this contract |
|---|---|---|
| First | Unsigned 32 bits | **A**, used in address calculations, bit extraction, and the entry invocation |
| Second | Opaque 32 bits | Not explicitly consumed by the represented body |
| Third | Unsigned 32 bits | **B**, forwarded at entry and used in a later destination calculation |

The unused second argument is not established as unobservable to dependencies. Unknown callees may consume machine state beyond the explicit argument lists shown.

Implicit inputs include:

- Incoming link state, which participates in selecting the first indirect target.
- Memory at every represented read location.
- Ordinary dependency results.
- An additional register result obtained after one dependency sequence.
- Execution-environment rules for addresses, alignment, permissions, and arithmetic.

### Output and completion

No explicit return value is represented. The final region has no local exit, so ordinary completion is not established.

Any dependency may fail, trap, not return, or transfer control elsewhere. No local error code, recovery path, rollback, or exception translation is represented.

### Widths and notation

A **word** is 32 bits; a **byte** is 8 bits. Unsigned right shifts are logical. A byte store supplies only the low eight bits of its value.

For the address and shift relationships below, **low32(v)** denotes retention of the low 32 bits. Arithmetic addresses are interpreted modulo \(2^{32}\). This is a conditional machine-word interpretation: some source expressions use signed intermediates, and the representation does not independently establish their executable overflow semantics.

The following notation describes locations, not dependencies or object layouts:

- **W(x)**: the word location at \(\operatorname{low32}(4x+20)\).
- **T(x)**: the word location at \(\operatorname{low32}(16x+20)\).
- **I(x)**: a byte destination obtained by freshly reading the unsigned byte at \(\operatorname{low32}(x+1)\), then adding 17.

Every use of I entails a new selector read. Its destination lies between numeric addresses 17 and 272 inclusive.

No allocation boundary, enclosing structure, endianness, memory-region type, or alignment-fault policy is established. A word transfer is a four-byte access, not a promise of atomicity.

## Behavior

### Entry and A-derived transfers

The first explicit effect is a word store of \(A \gg 7\) at W(A). It occurs before every represented dependency invocation.

The entry invocation receives A, A, B, and A, in that order. Its target is derived from incoming link state. Four subsequent indirect invocations lead to a result **r**, after which W(r) receives the same value \(A \gg 7\). This value is recomputed from A, not loaded from W(A). The next invocation receives low32(4r).

A following sequence produces a word **P**. Two byte transfers then use the selector location \((A \gg 8)+1\):

| Transfer | Byte supplied | Ordering |
|---|---|---|
| First | Low byte of \(P \ll 2\) | Selector read and store precede the next invocation |
| Second | Low byte of \((P\mathbin{\&}\text{0x3fffffff})\gg20\) | Occurs after intervening invocations and a fresh selector read |

The invocation immediately following the first transfer receives low32(\(P\ll2\)). The sequence between these transfers also produces a result whose low32 left shift by two is forwarded immediately after the second transfer.

The second stored byte is P’s bits 20–27. Although the masked right-shift expression can contain ten bits, the destination receives only eight.

The destinations can differ because the selector is reread. The first store itself may also affect a later read if addresses alias; dependency mutation is not the only possible source of change.

### Fixed-location snapshot

After further invocations, the representation reads a global associated with numeric address 60 and converts its value to unsigned 32 bits. Call the resulting saved value **G**. The global’s declared width is not supplied.

G is stored at W(\(B\gg7\)). The next invocation explicitly receives G. After additional invocations, G is stored at the same computed address again, without rereading the global.

The last result produced before that repeated store is subsequently forwarded after a low32 left shift by two. Changes made by dependencies to either the global or the destination do not replace the saved G used by the second store.

### Word snapshots and derived destinations

A later dependency supplies **X**. The function stores low32(4X) at T(X), then reads W(X) into saved word **H**. These accesses are ordered store-before-read.

Under the modular address convention, T(X) and W(X) coincide exactly when X is a multiple of \(2^{30}\). In that case, the snapshot reads the just-written word, subject to the memory environment and external interference.

The following invocation receives, in order:

- low32(16X);
- low32(4X);
- low32(\(X\ll26\));
- low32(4X).

Subsequent invocations forward intermediate dependency results. After those invocations, the function repeats the store of low32(4X) at T(X). A newly obtained result is then forwarded after a low32 left shift by two.

The remaining snapshot-related phases are ordered as follows. Each named result is obtained from a dependency at its own phase; it is not a reread of an earlier value.

| Result or phase | Memory effects and explicit argument relationships |
|---|---|
| **Y** | Store low32(4Y) at T(Y), then read W(Y) into saved word **J**. The next invocation receives low32(4Y) twice. After intervening calls, overwrite W(Y) with \(J\gg8\). |
| **Z** | Store low32(4Z) at T(Z). Begin the next call sequence with \(H\gg16\). After that sequence, store low32(4Z) at W(Z). |
| **Q** | Store low32(4Q) at T(Q). One invocation receives low32(16Q); a later invocation receives an intermediate result together with \(H\gg16\). Repeat the store of low32(4Q) at T(Q) afterward. |
| **R** | Store low32(4R) at T(R). One invocation receives low32(4R) and \(8(H\gg8)\); a later invocation receives an intermediate result together with \(H\gg16\). Afterward, store low32(4R) at address \(8(H\gg8)+20\). |
| **S** | Store low32(4S) at T(S). The next invocation receives \(8(H\gg8)\) twice, \(H\gg16\), and an X-derived field. After intervening calls, store \(8(H\gg8)\) at address \(8(H\gg8)+20\). |

The X-derived field in the S phase is:

\[
\operatorname{low32}(X\ll26)\gg21.
\]

It is formed through three successive logical right shifts by seven after the initial truncating left shift. Equivalently, under the stated width interpretation, it is \((X\mathbin{\&}63)\ll5\): X’s low six bits appear in positions 5–10.

H and J remain saved data dependencies. Replacing their later uses with fresh memory reads would change the represented behavior. Between these phases, newly produced results are also forwarded after low32 left shifts by two; such forwarding does not replace the named saved snapshots.

### Additional register result and later word transfers

After another invocation sequence, an additional register result **E** is used separately from the ordinary result path. Define **Vₑ** as low32(\(E\times2^{24}\)).

The function stores Vₑ at address low32(\(E\times2^{26}+20\)), then explicitly passes Vₑ to a dependency. A later ordinary result **U** selects W(U), where the same saved Vₑ is stored. The next invocation receives low32(4U).

Further phases follow:

| Result | Observable behavior |
|---|---|
| **V** | Store low32(4V) at T(V); begin a dependency sequence by passing low32(4V). A later result selects W(r), where the saved low32(4V) is written. |
| **N** | Store low32(4N) at T(N); begin a dependency sequence by passing low32(4N). A later result selects W(r), where \(H\gg24\) is written. |
| **M** | Store low32(4M) at T(M). Invoke a dependency with low32(16M), low32(16M), and \(H\gg23\). After further calls, write low32(16M) at T(M). |

The two M-derived stores supply different expressions: the second uses 16M, not 4M. Their values may coincide for particular inputs, but the second is not specified as a duplicate of the first.

For the following byte phases, define:

\[
K=(\operatorname{low32}(4N)\mathbin{\&}\text{0x7fffffff})\gg7.
\]

Thus K is between 0 and \(\text{0x00ffffff}\).

### Final byte-transfer phases

A dependency supplies **P₂**. The function reads the selector for I(K) and stores the low byte of \(P₂\ll2\). The next invocation receives low32(\(P₂\ll2\)) in three explicit argument positions.

After intervening invocations, a fresh I(K) selection receives the low byte of \((P₂\mathbin{\&}\text{0x3fffffff})\gg20\). A result from that intervening sequence is then forwarded after a low32 left shift by two.

A later dependency supplies **P₃**. Two byte stores follow without any intervening dependency invocation:

1. The low byte of \(P₃\ll2\) is stored at address \((H\gg31)+28\).
2. A fresh selector read for I(K) selects a destination that receives zero.

The direct destination is either 28 or 29. The second selector read follows the direct store, so aliasing can affect not only the second destination’s contents but also its selection.

Subsequent invocations explicitly receive \(H\gg31\), and later \((P₃\mathbin{\&}\text{0x3fffffff})\gg6\). After further invocations, another fresh I(K) selection receives zero.

Define **D** as low32(\(P₃\ll2\)). A later dependency supplies **P₄**, from which the function obtains:

\[
L=(P₄\mathbin{\&}\text{0x3fffffff})\gg25.
\]

L is between 0 and 31. Before entering the repeating tail, the function:

- Reads a byte at low32(\(1024D+1\)) and stores zero at that byte’s unsigned value plus 17.
- Invokes a dependency with \(2L\).
- Obtains an ordinary result **C** from the next invocation.

D remains the basis of the selector address throughout the repeating tail.

### Repeating tail and argument recurrences

If reached, the tail begins an iteration without an exit-condition check. Every iteration has this observable order:

1. An indirect invocation with two explicit arguments.
2. Four further indirect invocations, the last yielding an intermediate result **T**.
3. A fresh byte read at low32(\(1024D+1\)), followed by a zero byte store at the selected byte value plus 17.
4. An invocation receiving low32(\(T\ll2\)).
5. Four further indirect invocations, the last yielding a raw result **Rₙ** for that iteration.

The two arguments at the start of an iteration have distinct recurrences:

| Iteration | First argument | Second argument |
|---|---|---|
| First | C, unchanged from the ordinary result obtained immediately before the loop | \(L\gg8\), necessarily zero |
| Subsequent | low32(\(Rₙ\ll2\)), using the preceding iteration’s raw final result | \(\operatorname{low32}(Rₙ\ll26)\gg8\), using that same raw result |

Consequently, the raw final dependency result is **not** forwarded unchanged as the next iteration’s first argument. It is shifted left by two with 32-bit truncation.

The second argument is separately derived from the raw result, not from the already-shifted first argument. Its potentially nonzero bits occupy positions 18–23; equivalently, it is \((Rₙ\mathbin{\&}63)\ll18\).

The first iteration’s leading target is formed using a different represented continuation value from the target used on later iterations. The leading invocation must not be treated as one invariant callee across loop entry and repetition.

## Side effects

Explicit side effects consist of indirect invocations, word reads and stores, and byte reads and stores. The representation establishes no allocation, deallocation, ownership transfer, locking, or resource acquisition.

Dependencies are not assumed to preserve globals, selectors, destinations, or other reachable memory. Matching supplied store values do not establish unchanged memory between stores.

Observable ordering matters:

- The initial store precedes all dependency invocations.
- Snapshot-producing reads follow their associated stores.
- Later snapshot uses retain the saved values despite intervening calls.
- Selector reads occur afresh at each represented use.
- Consecutive byte operations remain distinct even without a call between them.
- Repeated tail iterations include both the selector read and the zero store.

Progress requires accessible memory and valid returning dependency targets. Alignment requirements depend on the unknown execution environment. Violations have no represented local recovery.

## Dependencies

Only F-97cface4b3d4 has a supplied identity mapping. The following opaque IDs label unresolved roles, not proven-distinct implementations:

| ID | Role |
|---|---|
| F-U001 | Entry invocation selected using incoming link state |
| F-U002 | Continuation-derived indirect invocations, including result-forwarding calls |
| F-U003 | Invocations supplying ordinary results used as data or address inputs |
| F-U004 | Invocation associated with the additional register result |
| F-U005 | Repeating-tail invocations |

These roles can overlap. Distinct call sites need not resolve to distinct functions. Explicitly argument-free invocations do not establish that the callee consumes no incoming machine state.

Target identities, complete calling conventions, memory effects, ownership behavior, failure behavior, and preservation rules remain unknown. No dependency is established as pure, an allocator, a validator, or a destructor.

## Acceptance examples

These are conditional static-contract examples, not executed tests. They assume the relevant accesses complete and dependencies return as needed.

1. **Initial boundary.** A = 127 supplies a zero word to address 528. A = 128 supplies a word value of one to address 532. Each store precedes the entry invocation.

2. **Fresh selectors.** With P = \(\text{0x12345678}\), the two A-derived byte values are \(\text{0xe0}\) and \(\text{0x23}\). Selector values of 3 and 9 at the respective reads select destinations 20 and 26.

3. **Saved snapshot.** If J = \(\text{0x89abcdef}\), the later overwrite of W(Y) supplies \(\text{0x0089abcd}\), even if a dependency changed W(Y) after the snapshot.

4. **H-derived fields.** If H = \(\text{0x81234567}\), the H-derived high-byte word store supplies \(\text{0x81}\), the M-phase third argument is \(\text{0x102}\), and the direct P₃-derived byte destination is 29.

5. **Consecutive-store alias.** If the direct destination is 28 and the following selector read yields 11, the direct P₃-derived byte at address 28 is immediately overwritten with zero.

6. **First iteration is unshifted.** If C = \(\text{0x40000001}\), the first leading tail invocation receives \(\text{0x40000001}\) as its first argument and zero as its second. The first argument is not four.

7. **Subsequent iteration is shifted.** If an iteration’s raw final result is \(\text{0x40000001}\), the next leading invocation receives \(\text{0x00000004}\) and \(\text{0x00040000}\). The first calculation discards overflow; the second independently retains the raw result’s low six bits.

8. **Different truncation boundaries.** If the raw final result is \(\text{0x80000040}\), the next leading arguments are \(\text{0x00000100}\) and zero. A nonzero next first argument does not imply a nonzero second argument.

9. **No finite-success criterion.** If accesses complete and all dependencies return, the represented tail continues repeating. A finite ordinary return is not an established successful outcome.

## Uncertainties

The following remain **documented_only**:

- Whether invalid instruction data or removed regions compromise represented operations or control-flow edges.
- Architecture, ABI, instruction-state conventions, endianness, and executable signed-overflow semantics.
- The fixed-location global’s declaration width and overlapping-storage interpretation.
- Whether numeric addresses designate ordinary memory, mapped resources, or something else.
- The origin, validity, and calling-convention meaning of the additional register result.
- Dependency identities, full input state, preservation guarantees, ownership, and errors.
- Atomicity, concurrency, volatile behavior, and externally triggered memory changes.
- Whether the unconditional repeating tail faithfully represents executable control flow.

Packed-field manipulation and address-indexed state updates are plausible inferred intent. They do not establish an application-level purpose or justify classifying the accesses as device-register operations.

## Evidence

Established static evidence consists of represented access widths, unsigned shift expressions, argument relationships, saved-value reuse, access ordering, and unconditional tail repetition.

In particular, the tail explicitly distinguishes the unshifted pre-loop ordinary result from the transformed next-iteration argument. After each iteration’s final dependency result, the first argument is formed by a left shift of two; the second follows a separate truncating left shift of 26 and logical right shift of eight.

Diagnostic warnings limit confidence in executable correspondence. No implementation inspection beyond the supplied representation, dependency inspection, runtime execution, or runtime tests have been performed. No snapshot hash or independent provenance assertion is included.
