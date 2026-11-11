<!-- Trusted provenance: F-5f215faa3cb8; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-5f215faa3cb8

## Purpose

F-5f215faa3cb8 is a ten-argument routine returning a 32-bit unsigned word. Its behavior is governed by global-driven dispatch. Depending on the selected states, it transforms working values, calls three dependencies, exposes two temporary storage regions, writes individual bytes through a selected pointer, and returns a separately maintained result.

**Established static evidence:** the supplied representation contains states 0–12 and a default case. Global values and dependency implementations are absent. The evidence therefore establishes conditional state behavior, not a fixed call sequence, successful completion, or termination.

**Inferred intent:** the byte transformations and temporary-region calls suggest processing controlled by a byte-sized value. No application-level interpretation is established.

This contract describes the observable operations and cross-state data dependencies without assuming that the dispatcher reduces to a conventional straight-line operation.

## Interface

### Arguments

| Position | Visible width and interpretation | Direct use |
|---|---|---|
| 1 | Unsigned 32-bit word | Saved at entry; copied into the primary working word by state 1. |
| 2 | Opaque 32-bit value | Saved at entry; no subsequent direct use is visible. |
| 3 | Signed 32-bit integer | Saved at entry; supplies one dimension working field. |
| 4 | Opaque 32-bit value | Saved at entry; supplies argument 6 of F-6d94c659ffc9. |
| 5 | Unsigned 32-bit word | Supplies a scaled argument field and the secondary working word. |
| 6 | Opaque 32-bit value | Supplies argument 5 of F-6d94c659ffc9. |
| 7 | Opaque 32-bit value | Supplies argument 1 of F-e038a8b6ff42. |
| 8 | Signed 32-bit integer | Supplies another dimension working field. |
| 9 | Signed 16-bit integer | Supplies the initial control-byte calculation. |
| 10 | Opaque 32-bit value | Supplies argument 3 of F-e038a8b6ff42. |

Opaque values are not established as pointers, handles, lengths, or ownership-bearing objects.

### Return and storage

The return is the **stored result**, a 32-bit word initialized to zero. Only state 5 directly updates it. Returning after dependency calls does not, by itself, return their results.

Two distinct temporary regions, **region A** and **region B**, each occupy 16 bytes. Neither is visibly initialized at entry. Both addresses can be passed to F-6d94c659ffc9; region B also reaches F-e038a8b6ff42 and can be selected for direct byte writes.

The routine performs no explicit heap allocation or deallocation. Temporary addresses are valid only within the invocation’s storage lifetime. Dependency ownership, retention, initialization, and access extents are unknown.

### Working-state terminology

The following names describe distinct storage roles, not implementation identifiers:

- **Primary** and **secondary:** 32-bit working words.
- **Scaled fifth argument:** a 32-bit field initially prepared from argument 5.
- **Dimension-three** and **dimension-eight:** 32-bit fields initially prepared from arguments 3 and 8; later states overwrite them with unrelated arithmetic.
- **Candidate**, **saved candidate**, and **intermediate result:** distinct 32-bit fields.
- **Counter**, **address word**, and **address difference:** distinct 32-bit fields.
- **Control byte:** held in 16-bit storage, but explicitly restricted to its low eight bits whenever assigned.
- **Write pointer:** initially null; state 8 and the default case select region B.
- Four saved dependency arguments, initially zero, later populated from arguments 4, 6, 7, and 10.

The original arguments 1–4 also have entry-time saved copies.

## Behavior

### Arithmetic and global reads

Calculations below describe 32-bit word results modulo \(2^{32}\), with explicit eight- and sixteen-bit truncations where shown. This is a word-level description, not a guarantee that every signed expression is portable across language or machine models.

Global symbols identify storage roles:

- \(K\): multiplier captured once at entry.
- \(M\): repeatedly read multiplier.
- \(D\): dispatcher bias.
- \(E\): counter/address arithmetic global.
- \(H\), \(J\): dependency-related multipliers.
- \(A,B,C,Q,R,S,T,U,V,W\): other arithmetic globals.
- \(Z\): candidate-source global.
- Transition globals: described by the state that reads them. States 0 and 9 share one; states 11 and 12 share another.

Every occurrence of a global denotes a separate read unless explicitly identified as a saved value. In particular, equal symbols do not guarantee equal observations. Each dispatcher iteration reads and saves \(M_s\) before reading the dispatcher bias. Dependency calls need not preserve globals.

For states 2, 4, and 10, \(t_o\) means the first transition-global read, used to construct the offset alternative; \(t_b\) means the subsequent read of that same global, used for the unadjusted alternative. Equality is not assumed.

### Entry and dispatch

Entry captures \(K\), then zero-initializes the counter, dependency-argument fields, stored result, intermediate result, saved candidate, dimensions, control storage, address word, and address difference. It initializes the write pointer to null.

The initial-selector global is loaded **before** arguments 1–4 are saved. Between that selector load and the argument saves, primary, secondary, and candidate are initialized to zero. Neither temporary region is initialized.

Each iteration:

1. Reads \(M\) into the iteration snapshot \(M_s\).
2. Reads \(D\) and dispatches on the current selector plus that bias.

A selector loaded by one state is therefore not necessarily the number of the next state. The dispatcher addition’s signed-overflow behavior is not independently established by the supplied declarations.

### State 0: replace candidate

The shared transition global is read and assigned to the selector **before** \(Z\) is read into candidate.

Stored result, primary, and secondary are unchanged.

### State 1: prepare working arguments

In visible assignment order, this state:

1. Saves argument 4 for dependency position 6.
2. Sets dimension-three to \(K\) times saved argument 3.
3. Saves argument 6 for dependency position 5.
4. Sets scaled fifth argument to a fresh \(M\) read times argument 5.
5. Sets dimension-eight to another fresh \(M\) read times argument 8.
6. Saves argument 7.
7. Sets control byte to \(\operatorname{low8}(189\cdot\text{argument 9})\).
8. Saves argument 10.
9. Reads its transition global into the selector.
10. Copies saved argument 1 into primary and argument 5 into secondary.

The two \(M\) reads need not agree. No guard requires this state before later dependency calls.

### State 2: first dependency and arithmetic branch

Before calling F-6d94c659ffc9:

- Secondary becomes \(K\) times the **incoming primary**.
- Control byte becomes \(\operatorname{low8}(149\cdot\text{control byte})\).
- Dimension-three becomes a fresh \(M\) read times its previous value.

Primary is not replaced by that initial scaling; secondary receives it.

The dependency receives these nine arguments, in positional order:

1. Scaled fifth argument.
2. Address of region A.
3. Control value in the range 0–255.
4. Address of region B.
5. Saved argument 6.
6. Saved argument 4.
7. Dimension-eight.
8. Updated dimension-three.
9. A fresh \(M\) read times secondary.

Let its consumed 32-bit result be \(r\). Using the iteration’s pre-call snapshot:

\[
p=M_sKr.
\]

Candidate becomes \(p\), and primary becomes \(Kp\). The branch bit is

\[
\beta=
\operatorname{highbit}\!\left(
\neg\left[
p\lor
\left((M_s\mathbin{\mathrm{ashr}}31)-M_s\right)Kr
\right]\right).
\]

The routine then reads its transition global to obtain \(t_o\), forming \(t_o+1\), and reads it again to obtain \(t_b\). Intermediate result receives primary.

The next selector is \(t_b\) when \(\beta=1\), otherwise \(t_o+1\). These alternatives need not share a common observed base.

### State 3: transition only

This state reads its transition global into the selector. It makes no direct dependency call or byte write.

### State 4: second dependency and constructed-byte branch

Primary becomes a read of \(H\) multiplied by secondary. F-cfd9a93ecce5 receives primary as its sole argument.

Only a signed 16-bit result \(s\) is retained. Control byte becomes

\[
\operatorname{low8}(16539s)\lor\operatorname{low8}(11877s).
\]

Its signed-eight-bit interpretation supplies the branch bit: one for negative, zero otherwise.

The transition global is read first as \(t_o\), constructing \(t_o+9\), and again as \(t_b\). Negative selects \(t_b\); nonnegative selects \(t_o+9\).

This is not a direct sign test of the original dependency result.

### State 5: capture the return value

Stored result becomes a fresh \(M\) read times intermediate result. The transition global is then read, after which primary receives stored result.

### State 6: restore a scaled candidate

Intermediate result becomes \(K\) times a fresh \(M\) read times saved candidate. The transition global is read, then primary receives intermediate result.

### State 7: return

The routine immediately returns stored result. No dependency result is implicitly substituted.

### State 8 and default: select the write region

Write pointer becomes the beginning of region B, followed by a transition-global read.

This neither initializes region B nor resets the counter or validates a future index.

### State 9: third dependency

F-e038a8b6ff42 receives:

1. Saved argument 7.
2. Address of region B.
3. Saved argument 10.

Its consumed 32-bit result is multiplied by \(K\), a post-call read of \(M\), and a read of \(J\). Primary receives that product. The shared transition global is read next, and candidate then receives primary.

Unless state 1 has populated them, the saved first and third arguments remain their initial zero values, subject to unknown indirect effects.

### State 10: arithmetic branch and persistent working-field updates

Let \(n\) be the incoming counter and \(a\) the address word. Define

\[
x=MKn,\qquad y=KnA+1.
\]

The visible assignments first place \(y\) in secondary, set address difference to \(a-E\), and set primary to \(x-1\).

Two separate reads of \(C\) participate in

\[
v=\left((x-(y\lor B)+C_{\mathrm{add}})
\mathbin{\oplus}\neg C_{\mathrm{complement}}\right)+1.
\]

Dimension-eight receives \(v\).

Three separate reads of \(R\) participate in

\[
w=
\neg\left((R_{\mathrm{right}}\gg1)
\lor(R_{\mathrm{left}}\ll31)\right)
-x+(y\lor R_{\mathrm{or}}).
\]

Scaled fifth argument receives \(w\). The shift combination must not be replaced by a rotate of one sampled value without an additional equality assumption.

Secondary is then overwritten with

\[
q=w\lor(x-1).
\]

Dimension-three receives

\[
z=v+
\left(
\neg((x-1)\lor Q)
\mathbin{\oplus}q
\mathbin{\oplus}\neg((x-1)\lor S)
\right).
\]

Primary becomes \(z-x\); its unsigned high bit is the branch bit.

The transition global is read first as \(t_o\), constructing \(t_o-8\), then as \(t_b\). Saved candidate receives candidate. High bit one selects \(t_b\); zero selects \(t_o-8\).

Thus the persistent arithmetic outputs are:

| Field | Value |
|---|---|
| Scaled fifth argument | \(w\) |
| Dimension-eight | \(v\) |
| Dimension-three | \(z\) |
| Secondary | \(q\) |
| Primary | \(z-x\) |
| Saved candidate | Incoming candidate |
| Address difference | \(a-E\) |

These values affect later states 2, 4, 6, and 11.

### State 11: update counter, write byte, and derive an address word

Counter first becomes

\[
n'=M(Kn+E).
\]

Dimension-eight becomes \(Kn'\).

From the incoming address word \(a\), secondary becomes

\[
b=K\left((aT-1)\lor\mathtt{0xfffffe8b}\right)+U+a,
\]

and dimension-three becomes

\[
c=V(a+W).
\]

A 16-bit retained product is formed from the signed interpretations of the low 16 bits of \(b\) and \(c\), multiplied by 149. Its low eight bits become control byte and the byte to be written.

The write occurs at write pointer plus the 32-bit product of a **fresh** \(M\) read and dimension-eight. There is no visible bounds check.

Only after this write, the routine forms an address-like value from the iteration’s earlier \(M_s\) times address difference. The shared address-selection tail then:

1. Sets address word to \(K\) times the 32-bit representation of that address-like value.
2. Reads the shared tail transition global.
3. Copies the new address word into primary.

Secondary remains \(b\), dimension-eight remains \(Kn'\), and dimension-three remains \(c\). These assignments are observable through subsequent states 2 and 4.

### State 12: seed address-derived state

This state selects the address of the counter, then sets the counter to 16. The shared tail sets address word to \(K\) times that counter address’s visible 32-bit representation, reads the tail transition global, and copies address word into primary.

Consequently, a subsequent state 2 scales this newly address-derived primary value. Address arithmetic can depend on invocation placement, not only on arguments.

## Side effects

Directly established effects are:

- Global reads, including repeated reads of the same storage.
- Conditional calls to three dependencies.
- Writes to invocation-local state.
- One byte write per visit to state 11.
- Exposure of temporary-region addresses.
- Conversion and arithmetic involving a temporary counter’s address.

No direct assignment to the listed globals is visible. Dependencies may nevertheless change globals or memory; their effects are not constrained by this representation.

For each reached state 11, write pointer plus computed index must designate writable storage. The routine does not establish that precondition:

- Write pointer starts null.
- Selecting region B establishes only its base address.
- The index is not restricted to 0–15.
- Counter update precedes index formation.
- Invalid memory access need not produce a normal return.

Callers and dependencies must also satisfy the dependencies’ unknown argument and storage requirements. No ownership transfer or release obligation can be inferred merely from pointer passage.

Separate statements establish ordering, including transition-before-candidate in state 0 and byte-write-before-address-tail in state 11. Relative evaluation order inside compound expressions and dependency argument lists is not established.

## Dependencies

| Opaque function ID | Visible interface | Unknown contract |
|---|---|---|
| F-6d94c659ffc9 | Nine arguments; receives both region pointers and an eight-bit-range control value; result consumed as 32 bits. | Meaning, access extents, initialization, errors, retention, ownership, global effects. |
| F-cfd9a93ecce5 | One computed 32-bit argument; result retained as signed 16 bits. | Meaning, result domain, errors, memory and global effects. |
| F-e038a8b6ff42 | Three arguments; middle argument points to region B; result consumed as 32 bits. | Buffer interpretation, initialization requirements, access extents, errors, retention, ownership, global effects. |

No explicit error propagation policy is established. Zero or negative-looking results undergo arithmetic and branching rather than automatic error return.

## Acceptance examples

These are conditional static criteria, not executed tests.

1. **Immediate exit:** if the first dispatch selects state 7, the return is zero and no dependency is called.

2. **Control-byte recovery:** state 1 followed by state 2 without an intervening control-byte change passes \(\operatorname{low8}(\text{argument 9})\) in dependency position 3, because \(189\cdot149\equiv1\pmod{256}\). It does not recover the full signed 16-bit argument.

3. **Primary-to-secondary transfer:** entering state 2 with primary \(P\) makes secondary \(KP\) before the call. Its ninth argument is a fresh \(M\) observation times \(KP\), not necessarily a scaling of the previous secondary.

4. **State-10-to-state-2 propagation:** absent intervening changes, state 2 receives \(w\) in position 1 and \(v\) in position 7; position 8 is a fresh \(M\) read times \(z\). Its ninth argument derives from \(K(z-x)\).

5. **State-10-to-state-4 propagation:** absent intervening changes, state 4 calls F-cfd9a93ecce5 with \(Hq\), not \(Hy\).

6. **State-11 persistence:** after a valid state-11 write, a subsequent unchanged state 4 uses \(Hb\). A subsequent state 2 consumes dimension-eight \(Kn'\), rescales dimension-three \(c\), and scales the address-derived primary.

7. **Distinct transition observations:** in state 4, a constructed zero byte selects the first observed transition value plus 9. A negative constructed byte selects the second observed value. Equality of those observations is unnecessary.

8. **Constructed-byte sign:** a retained state-4 result with low byte 1 produces \(155\lor101=255\), selecting the negative-byte alternative under the shown signed-byte interpretation.

9. **Return isolation:** state 9 followed by exit, with no state 5 visit, returns zero regardless of the dependency result.

10. **Address-tail propagation:** state 12 establishes counter 16 and makes both address word and primary equal to the scaled counter address. A later state-11 write index is not necessarily 16 because that state first updates and scales the counter.

11. **Unsafe write path:** reaching state 11 with the initial null write pointer produces no visible validation error before the attempted write. Normal completion is not guaranteed.

## Uncertainties

The following remain **documented_only**:

- Global values, declarations, mutability, and synchronization requirements.
- Actual state reachability, repetition, and termination.
- Equality of separate reads, including transition globals, \(C\), \(R\), and \(M\).
- Evaluation order among operands within compound expressions.
- Application-level meaning of arguments, buffers, and results.
- Dependency behavior, including ownership, pointer retention, memory access, and global changes.
- Whether either temporary region is initialized before a dependency reads it.
- Whether computed write addresses are valid.
- Machine details governing signed overflow, signed-byte conversion, pointer conversion, and address indexing.
- Exact shift semantics requiring omitted declarations: state 2 visibly uses a signed working value for its right shift; state 10’s global right-shift operand lacks a supplied declaration.

The specification preserves separate state-10 shift operands rather than assuming a single rotate. It does not generalize the visible 32-bit address arithmetic to a wider-pointer environment.

## Evidence

This contract derives from static inspection of the supplied routine representation and opaque dependency mapping. The prior draft and feedback are not independent evidence.

Established observations include argument widths, two 16-byte regions, entry initialization order, dispatcher structure, dependency argument positions, persistent cross-state assignments, repeated transition and arithmetic-global reads, byte-write ordering, address-tail assignments, and the stored-result return.

No global definitions, dependency implementations, execution traces, or runtime observations were supplied. **No runtime tests have been performed.**
