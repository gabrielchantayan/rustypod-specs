<!-- Trusted provenance: F-cbfb908a0c47; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-cbfb908a0c47

## Purpose

F-cbfb908a0c47 conditionally delegates an operation on a state object. An enablement bit controls entry. A preliminary dependency selects either an alternate delegation or a maintenance path.

The maintenance path may replace a backing region, reconstruct three address-like fields, conditionally rebase three additional fields, and delegate with an input restricted to its low eight bits.

**Established static behavior:** the accesses, local initializations, arithmetic, branches, dependency arguments, direct writes, and return selection described below.

**Inferred intent:** the state resembles a buffered object with replaceable storage and multiple cursor groups. Allocation, transfer, release, and byte-output terminology describes apparent roles, not established dependency semantics.

**Unknown:** application-level meanings and complete dependency contracts. In particular, no dependency is assumed to preserve globals or state, allocate usable memory, or release owned memory.

## Interface

### Inputs and result

The function receives:

- A **state address**, represented by a 32-bit signed address-carrying value.
- An **input word**, represented by an unsigned 32-bit value.

It returns an uninterpreted 32-bit word.

| Condition | Returned word |
|---|---|
| Enablement bit `0x8` at state offset `0x04` is clear | `0xffffffff` |
| Enabled; F-d52d006afc42 returns nonzero | Result of F-9916da3e9ea8 |
| Enabled; F-d52d006afc42 returns zero; maintenance reaches final delegation | Result of F-d34766aa836c |

Returned dependency results undergo no visible normalization. The initial rejection word does not establish a shared error convention.

### Directly accessed state

All listed fields occupy four bytes. Offsets are byte offsets from the state address. Labels describe visible uses, not established application types.

| Offset | Visible use |
|---|---|
| `0x04` | Unsigned flag word; bit `0x8` enables processing |
| `0x08` | Backing address, read as a signed address-like word |
| `0x0c` | Unsigned extent |
| `0x10` | Unsigned flag word; bit `0x1` gates a release-like call |
| `0x14` | Old base for secondary reconstruction |
| `0x18` | Address-like field rebased relative to the old `0x14` |
| `0x1c` | Word conditionally replaced from `0x28` |
| `0x20` | Address-like field assigned from `0x08` |
| `0x24` | Address-like field assigned backing address plus saved displacement |
| `0x28` | Unsigned comparison operand; later assigned the successor of `0x24` |

No direct access to offset `0x00` is visible. Dependencies may access other fields or unrelated memory.

### Enabled-path local storage

Before the preliminary call, the function initializes two separate four-byte local cells, in this order:

1. A saved-state cell receives the state address.
2. An input cell receives the full input word.

The supplied stack representation places the saved-state cell immediately before the input cell, at the preceding four-byte position. This is evidence about the represented local layout, not a portable aggregate layout, public record format, or established argument block.

F-9916da3e9ea8 receives the address of the input cell, not an explicit address of the saved-state cell. The caller shows no subsequent explicit read of the saved-state cell. Nevertheless, its initialization and adjacency remain relevant because the dependency’s access extent is unknown. Whether that dependency accesses the preceding saved-state cell is **documented_only**.

A separate four-byte local temporary is passed to F-6c292b887399 on one replacement subpath. No initialization of that temporary is visible before the call.

### Preconditions and representation

Every reached access requires appropriate readable or writable storage. There is no explicit validation of the state address, alignment, field accessibility, or address range.

Unsigned arithmetic described below uses 32-bit words and wraps modulo \(2^{32}\). Signed address reconstruction is distinguished from those unsigned calculations; behavior outside the signed representable range is **documented_only**, not specified here as wrapping.

Dependency-specific memory requirements remain external preconditions. A replacement address is used without a visible validity check.

## Behavior

### Enablement, local initialization, and preliminary dispatch

The first visible state access reads `0x04`.

If bit `0x8` is clear, the function returns `0xffffffff`. It performs no listed dependency call, no direct state write, and neither of the enabled-path local initializations.

If the bit is set, the function initializes the saved-state cell and then the input cell as described above. It next calls F-d52d006afc42 with the state address and full unsigned input word.

Only zero versus nonzero is tested in the result.

For a nonzero result, the function calls F-9916da3e9ea8 with the input cell’s address and returns that dependency’s result. There is no masking before this call and no explicit inspection of either initialized cell afterward.

This path performs no maintenance and does not call F-d34766aa836c. Dependency effects are not rolled back. Access beyond the input cell, modification of local storage, and retention of its address are not established by the visible call.

For a zero result, execution continues with maintenance.

### Saved displacement and replacement decision

The function calls F-14a841015476 with the state address and saves its 32-bit signed result as a displacement.

After that call, it reads the unsigned extent at `0x0c`; call this saved value **E**. It then compares the wrapped sum of the backing address and **E** against the unsigned word at `0x28`.

For the backing-address read **B** and comparison-limit read **L**, replacement is selected exactly when:

\[
((B + E) \bmod 2^{32}) \leq_{\mathrm{unsigned}} L.
\]

Although the backing address is read through a signed representation, the unsigned extent makes this addition and comparison unsigned. Equality selects replacement. Overflow does not reject replacement; the comparison uses the wrapped sum.

The extent read precedes the comparison. No total order is established between the backing-address read and limit read within that comparison.

Both preceding dependencies have already run, so their changes to these fields can affect the decision.

If replacement is not selected, all replacement-specific calls and installation writes are skipped. Primary reconstruction still follows.

### Replacement extent

When replacement is selected, the function derives the new extent from saved **E**, not a later extent read.

First calculate:

\[
C =
\left(E+\left\lfloor E/2\right\rfloor+
\left\lfloor E/8\right\rfloor\right)\bmod 2^{32}.
\]

The divisions are unsigned right shifts.

Compare **C** with \((E+32)\bmod 2^{32}\) as unsigned words. If **C** is smaller, replace it with that wrapped increment.

Next compare \((E+1)\bmod 2^{32}\) and **C** after interpreting each as signed 32-bit:

- If signed \(E+1\) is strictly less than signed **C**, select **C**.
- Otherwise select the wrapped word \(E+1\).

The selected bit pattern becomes the unsigned replacement extent.

This is not an unbounded mathematical maximum. Unsigned wrapping followed by signed comparison can produce an extent smaller than **E**, including zero. No explicit overflow, shrinkage, or zero-size rejection is visible.

### Replacement acquisition and conditional calls

The function calls F-871e64ddc6de with the replacement extent and zero. It retains the returned 32-bit word as the prospective replacement address.

After this call, it rereads `0x08` and tests that backing address for zero.

If zero:

- F-external-01 is skipped.
- F-6c292b887399 is skipped.
- The release-gating flag at `0x10` is not read for this conditional activity.

If nonzero, the function calls F-external-01 with:

1. The retained replacement address.
2. A fresh read of the current backing address at `0x08`.
3. A fresh read of the current extent at `0x0c`.

F-external-01 is an opaque identifier for the external dependency. Its argument pattern suggests transfer, but copying, overlap handling, and failure semantics are unknown.

The backing-address argument is a separate read from the preceding nonzero check. The extent argument need not equal saved **E** or the replacement extent. No order is established between these independent argument reads.

After F-external-01 returns, the function reads `0x10`. If bit `0x1` is set, it calls F-6c292b887399 with the address of the separate four-byte local temporary and another fresh read of `0x08`.

That temporary is not visibly initialized before this call or consumed afterward. Whether the dependency reads it, writes it, or accesses surrounding storage remains unknown.

### Replacement installation

After the conditional activity, the function calls F-cf4f89824645 with the state address and `1`.

On normal return, it performs these direct writes in order:

1. Store the retained replacement address at `0x08`.
2. Store the selected replacement extent at `0x0c`.

These stores overwrite any intervening dependency changes to those two fields.

Other dependency changes are not generally preserved: subsequent reconstruction writes can overwrite additional fields. No rollback mechanism or recovery copy of the prior backing address is visible.

### Primary reconstruction

Every maintenance path reaching this stage performs primary reconstruction, whether or not replacement occurred.

The function first reads `0x08` and adds the saved displacement using signed address arithmetic. It then performs three direct writes in order:

1. Write `0x20` from another read of `0x08`.
2. Write `0x24` with the constructed address.
3. Write `0x28` with the constructed address plus one.

The backing read used for construction precedes the separate backing read supplying `0x20`. The saved displacement is not recomputed after replacement-related dependencies.

For stable reads and representable signed arithmetic, `0x24` becomes backing plus displacement, and `0x28` becomes its one-unit successor. No application-level unit beyond the represented increment is established.

### Conditional secondary reconstruction

After the primary writes, the function calls F-7febad47082c with the state address.

A zero result skips the remaining direct reconstruction writes.

A nonzero result causes these ordered operations:

1. Read and save the old word at `0x14`.
2. Write `0x14` from the current backing address at `0x08`.
3. Write `0x18` with its current value minus the saved old `0x14`, then plus a fresh read of `0x08`.
4. Write `0x1c` from the current word at `0x28`.

The subtraction and addition are signed expressions; intermediate as well as final representability matters.

With stable backing reads and nonoverflowing arithmetic, this preserves the displacement of `0x18` from the old base while replacing that base.

F-7febad47082c runs before these reads. Its changes therefore affect reconstruction. In particular, `0x1c` receives the value of `0x28` after that dependency, not necessarily the value produced by primary reconstruction.

### Final delegation

The function calls F-d34766aa836c with the state address and the input cell’s word masked by `0xff`.

The second argument is a 32-bit value from 0 through 255, with its upper 24 bits zero. The caller shows no explicit write to the input cell after its initialization; this does not establish unrestricted dependency preservation guarantees.

The final dependency’s result is returned without visible conversion, subsequent direct state writes, rollback, or additional cleanup.

## Side effects

Direct state writes are confined to maintenance:

- Replacement installation: `0x08`, then `0x0c`, only when replacement is selected.
- Primary reconstruction: `0x20`, `0x24`, then `0x28`.
- Secondary reconstruction: `0x14`, `0x18`, then `0x1c`, only after a nonzero F-7febad47082c result.

Enabled execution also initializes the separate saved-state and input local cells before preliminary dispatch. Their visible ordering and represented adjacency do not prove how a dependency uses them.

The principal call ordering on a replacement path is acquisition, optional transfer-like activity, optional release-like activity, F-cf4f89824645, installation, primary reconstruction, F-7febad47082c, optional secondary reconstruction, and final delegation.

Dependencies may introduce additional memory, state, or global effects. None is assumed to preserve globals, preserve state, own or release storage, or return normally.

The function has no visible allocation-result validation, dependency-error recovery, or rollback. A failure-indicating result from final delegation, if defined by that dependency, follows the preceding effects.

## Dependencies

| Opaque ID | Visible interface and use |
|---|---|
| F-d52d006afc42 | State address and full input word; zero selects maintenance, nonzero selects alternate delegation |
| F-9916da3e9ea8 | Input-cell address; result returned directly; adjacent saved-state-cell access unresolved |
| F-14a841015476 | State address; result retained as signed displacement |
| F-871e64ddc6de | Replacement extent and zero; result used as replacement address |
| F-external-01 | Replacement address, current backing address, current extent |
| F-6c292b887399 | Four-byte local temporary address and current backing address |
| F-cf4f89824645 | State address and `1`, before replacement installation |
| F-7febad47082c | State address; nonzero enables secondary reconstruction |
| F-d34766aa836c | State address and masked input-cell value; result returned directly |

Complete dependency contracts are **documented_only**. Results other than those explicitly tested, retained, or returned have no further visible use here.

## Acceptance examples

These are conditional static examples, not executed tests. Unless specified otherwise, dependencies return normally without modifying relevant state or local cells, and signed address calculations remain representable.

1. **Disabled entry.** With `0x04` equal to zero, the function returns `0xffffffff`. No listed dependency runs, no direct state write occurs, and the enabled-path saved-state and input cells are not initialized.

2. **Initialized context on alternate delegation.** For state address `0x1000`, input `0x123456ab`, and a nonzero preliminary result, the saved-state cell is first initialized to `0x00001000`, followed by input-cell initialization to `0x123456ab`, before F-d52d006afc42 runs. F-9916da3e9ea8 later receives the input-cell address. In the supplied stack representation, the saved-state cell occupies the immediately preceding four bytes. This does not establish that the dependency reads those bytes.

3. **No replacement.** With backing `0x1000`, extent `64`, and `0x28` equal to `0x103f`, replacement is skipped. A saved displacement of `10` produces primary values `0x1000`, `0x100a`, and `0x100b`.

4. **Equality selects replacement.** With backing `0x1000`, extent `64`, and limit `0x1040`, replacement occurs. The initial growth candidate is `104`, the increment-by-32 candidate is `96`, and the increment-by-one candidate is `65`. The selected extent is `104`.

5. **Zero extent and absent old backing.** If replacement is selected with extent zero, the selected extent is `32`. If the backing-address check after acquisition observes zero, both conditional transfer-like and release-like calls are skipped.

6. **Mixed arithmetic boundary.** For extent `0xffffffff`, selected replacement produces an initial candidate of `0x9ffffffd`. The wrapped increment-by-32 is `31`, so the unsigned comparison retains that initial candidate. The wrapped increment-by-one is zero. Its signed comparison against negative `0x9ffffffd` selects zero, which reaches F-871e64ddc6de without visible rejection.

7. **Overwriting dependency changes.** If F-cf4f89824645 changes `0x08`, `0x0c`, and the primary fields, installation overwrites its first two changes with retained replacement values. Primary reconstruction then overwrites `0x20`, `0x24`, and `0x28`.

8. **Secondary rebasing.** If F-7febad47082c returns nonzero and leaves backing `0x2000`, old base `0x1000`, and `0x18` equal to `0x1012`, reconstruction writes `0x14 = 0x2000` and `0x18 = 0x2012`. It copies the then-current `0x28` into `0x1c`. A zero result skips all three writes.

9. **Final byte restriction.** With an unchanged input cell containing `0x123456ab`, maintenance passes `0x000000ab` to F-d34766aa836c. Its result is returned without rollback.

## Uncertainties

The following remain **documented_only**:

- Application-level meanings of flags, fields, results, and address increments.
- Complete dependency behavior, side effects, preservation guarantees, and nonlocal exits.
- Whether F-9916da3e9ea8 accesses the adjacent saved-state cell, modifies surrounding local storage, or retains the supplied address.
- Portability of the represented local adjacency; no aggregate-layout contract is established.
- Access requirements for the uninitialized four-byte temporary.
- Allocation success signaling, zero-size behavior, and suitability of returned storage.
- Transfer semantics, overlap handling, ownership, release behavior, and storage lifetimes.
- Valid displacement range, signed overflow behavior, alignment, byte order, and invalid-address behavior.
- Concurrency, reentrancy, and external mutation guarantees.

## Evidence

This contract rests solely on static inspection of the supplied function representation and opaque dependency mapping. The prior draft and reviewer feedback are not independent evidence.

The representation establishes the enabled-path saved-state initialization before input-cell initialization and preliminary dispatch, and places those separate four-byte cells adjacently in the stated order. It does not establish a portable aggregate or the alternate dependency’s access extent.

Other established claims concern visible widths, accesses, arithmetic distinctions, branches, call arguments, sequencing, writes, and result selection. Allocation, transfer, release, buffering, and cursor terminology expresses inferred roles only.

No runtime tests have been performed. Acceptance examples are conditional deductions, not observed executions.
