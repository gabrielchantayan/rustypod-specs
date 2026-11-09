<!-- Trusted provenance: F-7edad8af3735; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-7edad8af3735

## Purpose

F-7edad8af3735 is a globally controlled state-dispatch routine. Depending on the states reached, it can inspect a caller-provided 32-bit word, pass that word’s value to F-943e4a0efcb9, clear the word after the call, pass a retained address to F-5e3309c9d083, and perform calculated byte writes.

These are possible actions, not an established unconditional sequence. Global contents are unavailable, so the evidence does not establish a normal entry-to-return path, termination, or effective protection against null dereferences.

The value-based call followed by clearing resembles resource disposal or handle reset. That is **inferred intent**, not an established ownership contract.

The reported return is 64 bits. Its visible construction fixes the low 32 bits at zero and obtains the high 32 bits from a local slot with no visible initialization. No meaningful success or error result is established.

## Interface

### Input and retained addresses

The sole argument is nominally a pointer to a 32-bit integer. The routine maintains three distinct retained addresses:

- **Word pointer:** initially null; replaced with the original argument by the pointer-test action.
- **Address-call pointer:** initially null; copied from the word pointer by the word-test action.
- **Byte-write base:** initially null; copied from the address-call pointer by the arithmetic-selector action.

These addresses need not all become equal on an arbitrary dispatch path. In particular, reaching one action does not prove the actions that initialize its required address have occurred.

The word pointer is used for 32-bit reads and clearing. The address-call pointer is passed to F-5e3309c9d083. The byte-write base participates in calculated byte destinations. The word’s stored value is passed to F-943e4a0efcb9; it is not visibly used as the byte-write base.

### Preconditions and ownership

Preconditions are action-dependent:

- A reached word read requires readable 32-bit storage at the word pointer.
- A reached word clear requires that storage to remain writable after F-943e4a0efcb9 returns.
- A reached byte store requires its calculated destination to be writable.
- Dependency calls require whatever conditions their unavailable contracts impose.

There is no supplied length, allocation extent, alignment guarantee, ownership annotation, or bounds check. Null is not established as safe merely because one action tests for it.

The routine itself establishes neither ownership transfer nor release of an allocation. Either dependency could affect storage lifetime; neither may be assumed to preserve it.

### Result and arithmetic widths

On the visible return path:

- Result bits 0–31 are zero.
- Result bits 32–63 come from a 32-bit slot with no visible initialization.

The complete result is therefore not a supported deterministic status value. Whether the upper half is genuinely unspecified at machine level or reflects an incomplete recovered interface is **documented_only**.

The representation uses 32-bit state and arithmetic, 8-bit stores, character-sized narrowing, and a 64-bit return construction. Equations below describe low-32-bit arithmetic unless stated otherwise. They express a machine-width interpretation, not a guarantee that overflowing signed expressions have defined high-level-language semantics.

Calculated addresses pass through a 32-bit integer representation. Extending this contract to a 64-bit address model is unsupported.

## Behavior

### Entry state and global-read order

The significant entry reads occur in this order:

1. Read the global scale and retain it for the invocation.
2. Read the global entry-state location and retain that observation minus seven as the fallback state.
3. Read the same entry-state location again to initialize the current state.

The two entry-state observations are distinct. If their values differ, the fallback is based on the earlier observation, not the observation used for the initial current state.

The counter and both working arithmetic values initially equal zero. The scale does **not** initially equal zero by construction; it equals the first global observation above. The retained selector also initially equals zero. All three retained pointers initially equal null.

The cached scale and fallback are not refreshed by later reads of their original locations. Other globals are read when their expressions execute.

### Dispatcher

Each dispatch obtains a key from the 32-bit difference between the current state and a freshly read global baseline.

| Key | Action |
|---|---|
| 0 | Return the reported 64-bit result. |
| 1 | Read the dispatch baseline into the current state. |
| 2, or any unlisted key | Call F-5e3309c9d083, then select the cached fallback. |
| 3 | Select a state using the retained selector narrowed to character width. |
| 4 | Update the counter, prepare a working-state intermediate, and write one byte. |
| 5 | Call F-943e4a0efcb9 with the current word, then clear the word. |
| 6 | Calculate an arithmetic selector and capture the byte-write base. |
| 7 | Read a dedicated global destination into the state. |
| 8 | Test the original argument for null and retain it as the word pointer. |
| 9 | Initialize the counter and primary working value. |
| 10 | Read another dedicated global destination into the state. |
| 11 | Test the retained word and capture the address-call pointer. |

The table identifies actions, not their reachability or normal ordering. Each subsequent dispatch rereads the baseline. Thus key 1 does not guarantee key 0 next, even though it assigns a baseline observation to the state.

### Pointer and word tests

The pointer-test action:

- Sets the selector to one for a non-null original argument, otherwise zero.
- Reads its global next-state destination.
- Retains the original argument as the word pointer.

It does not dereference the argument.

The word-test action first reads the 32-bit word through the word pointer. It sets the selector to one for a nonzero word, otherwise zero. Its destination expression contains **two occurrences of the same destination global**.

Let \(D_0\) denote the observation in the term used for a zero word, and \(D_1\) the observation in the term used for a nonzero word. For Boolean selector \(b\), the selected state is

\[
(D_0+5)(b\mathbin{\mathrm{XOR}}1)+D_1b.
\]

Consequently:

- A zero word selects \(D_0+5\).
- A nonzero word selects \(D_1\).

Both global occurrences remain represented. Their operand-evaluation order is not established; neither is a single shared snapshot. Only if the global remains stable can these destinations be described using one value.

After that state expression, the action copies the word pointer into the address-call pointer. There is no local null guard around the word read.

### Selector-based state selection

The key-3 action narrows the retained selector to character width and promotes it for arithmetic. Let \(q\) be that promoted value, \(f\) the cached fallback, and \(D\) this action’s global destination observation. The state becomes

\[
f(q\mathbin{\mathrm{XOR}}1)+Dq.
\]

For \(q=0\), this selects the fallback. For \(q=1\), it selects the global destination. Other values produce arithmetic combinations rather than a Boolean choice. Character signedness matters if narrowing produces a value outside the nonnegative signed-character range.

### Value-based dependency call and clearing

The key-5 action has this order:

1. Read the 32-bit word through the word pointer.
2. Call F-943e4a0efcb9 with that value.
3. If the call returns normally, overwrite the word with zero.
4. Read the action’s global next-state destination.

There is no nonzero test inside this action. It can pass zero if dispatch reaches it with a zero word.

If the dependency changes the word while leaving its storage writable, the subsequent clear overwrites that change. If the dependency invalidates the storage, the representation still attempts the post-call store; safety is not established. An abnormal or nonreturning call prevents the visible continuation.

### Address-based dependency call

Key 2 and every unlisted key call F-5e3309c9d083 with the address-call pointer. After normal return, the state becomes the cached fallback.

The argument may still be null, or may be an address captured by a prior word-test action. It is not the stored 32-bit word. The fallback is not recomputed after the call, even if the dependency changes the entry-state global.

### Counter initialization

The key-9 action sets the counter to 40 and the primary working value to

\[
-44s,
\]

where \(s\) is the cached scale. It then reads the shared next-state destination also used after a byte-write action.

This action does not reset the second working value.

### Byte-writing action

Use the following mathematical names:

- \(s\): cached scale.
- \(c\): counter before the action.
- \(v\): primary working value before the action.
- \(w\): second working value.
- \(p\): captured byte-write base.

The action contains repeated reads of a multiplier global and an additive-parameter global. They must not be collapsed into one snapshot without a stability assumption.

Its observable sequencing is:

1. **Counter update.** Read multiplier occurrence \(M_c\) and additive occurrence \(K_c\), producing

   \[
   c'=M_c(sc+K_c).
   \]

   The counter is updated before the subsequent expression.

2. **Pending working-state computation.** Read the multiplier again, as \(M_v\), and the additive parameter again, as \(K_v\). Retain

   \[
   r=M_v(v-K_v).
   \]

   This computation uses the old primary working value.

3. **Byte store.** The address expression reads the multiplier a third time, as \(M_a\). The destination is the 32-bit address sum

   \[
   p+M_as c'.
   \]

   The stored byte is the low eight bits of

   \[
   (v+15)\cdot41\cdot
   \left(s\left((w-1)\mathbin{\mathrm{OR}}139\right)-91+v\right)
   \cdot89.
   \]

   The represented expression narrows the participating working values and scale to character width. The equation preserves the low-eight-bit store result without assigning them a wider signed interpretation.

4. **Working-state update.** After the store, replace the primary working value with

   \[
   sr.
   \]

5. **Next state.** Read the shared next-state destination.

No operand-evaluation order is asserted between the multiplier and additive reads within either individual multiplication expression. The separate expressions above do establish their sequencing relative to the store.

A byte store overlapping the multiplier or additive global cannot retroactively alter the already retained \(r\). A store overlapping the next-state global can affect the subsequent next-state read. A store overlapping the scale global does not replace the cached scale.

Only under stability of the relevant globals throughout the action may the counter and working-state equations use one common multiplier and additive value.

No fixed write count, contiguous traversal, allocation extent, or terminating condition follows from this action.

### Arithmetic-selector action

This action first forms an offset

\[
x=M_xsc,
\]

using a multiplier-global observation \(M_x\). It then forms

\[
y=scN+1,
\]

using a different global multiplier \(N\), and derives \(t=x-1\). Next, it replaces the second working value with \(Pv\), using another global multiplier \(P\).

The sign-derived selector depends on several additional global arithmetic and mask parameters. The represented relation can be expressed as

\[
\begin{aligned}
z={}&
\left(
  \left((x-(y\mathbin{\mathrm{OR}}A))-B\right)
  \mathbin{\mathrm{XOR}}C
\right)+1\\
&+\left[
 \neg(t\mathbin{\mathrm{OR}}D)
 \mathbin{\mathrm{XOR}}
 \left(
   \left((\neg(H_s\mathbin{\mathrm{RSHIFT}}1)-x)
          +(y\mathbin{\mathrm{OR}}H_o)\right)
   \mathbin{\mathrm{OR}}t
 \right)
 \mathbin{\mathrm{XOR}}
 \neg(t\mathbin{\mathrm{OR}}J)
\right]-x.
\end{aligned}
\]

Here \(A,B,C,D,J\) are distinct global roles. \(H_s\) and \(H_o\) are separate occurrences of the same global: one participates in a right shift, the other in an OR. No relative evaluation order or equality of those observations is established. Complements and bitwise operations have 32-bit width.

The selector is the negation of the signed interpretation of \(z\) shifted right by 31. Under arithmetic signed-right-shift semantics, it is zero for a nonnegative signed \(z\) and one for a negative signed \(z\).

The following destination expression again contains two occurrences of one global. Calling their observations \(T_0\) and \(T_1\), the state is

\[
(T_0+3)(b\mathbin{\mathrm{XOR}}1)+bT_1,
\]

where \(b\) is the calculated selector. For Boolean \(b\), zero selects \(T_0+3\), and one selects \(T_1\). Both occurrences remain present; their evaluation order is not established.

Finally, the action copies the address-call pointer into the byte-write base.

This arithmetic is not established to be a bounds check, count comparison, or conventional loop condition. The semantics of the global’s own right shift and any simplification of the predicate remain unresolved.

## Side effects

Possible direct effects are word reads, post-call word clearing, calculated single-byte stores, and repeated global reads. Calls may introduce additional effects.

No explicit direct assignment to a named global is present. Global preservation nevertheless does not follow:

- Dependencies may modify globals.
- Caller-associated storage may alias globals.
- Calculated byte destinations may overlap globals.

Later reads can observe such changes. Previously cached values and retained arithmetic intermediates are not refreshed merely because their original storage changes.

Dispatch may repeat calls, clears, tests, or byte stores. Neither once-only execution nor termination is established.

There is no visible error-reporting branch, rollback, or handler converting invalid accesses or dependency failures into a defined return status. The evidence also supplies no synchronization contract; distinct global occurrences do not establish that concurrent mutation is supported.

## Dependencies

| Opaque function ID | Established invocation | Unresolved contract |
|---|---|---|
| F-943e4a0efcb9 | One argument containing a 32-bit value read through the word pointer. | Value interpretation, ownership, storage lifetime, global effects, failure behavior, and normal return. |
| F-5e3309c9d083 | One argument containing the address-call pointer, initially null and later possibly captured from the word pointer. | Null acceptance, ownership, memory effects, global effects, failure behavior, and normal return. |

Neither dependency’s return value is visibly used. Neither may be assumed to allocate, release, preserve, or refrain from releasing memory.

## Acceptance examples

These are conditional static expectations, not executed tests.

### Different entry-state observations

**Condition:** The scale read yields 3. The first entry-state read yields 100; the second yields 200.

**Expected:** The cached scale is 3, the fallback is 93, and the initial state is 200. The counter and both working values still initially equal zero.

### Null pointer test

**Condition:** Dispatch reaches key 8 with a null argument.

**Expected:** The selector becomes zero and the word pointer becomes null. This action performs no word read.

**Not established:** Safe return or protection against a later null dereference.

### Repeated word-destination reads

**Condition:** Key 11 reads a zero word. The destination occurrence in the zero-word term observes 100, while the other occurrence observes 900.

**Expected:** The selected state is 105. For a nonzero word with those same occurrence-specific observations, it would be 900.

No ordering between the two observations is implied.

### Call followed by clear

**Condition:** Key 5 reads value 17, and F-943e4a0efcb9 returns normally while leaving the word writable.

**Expected:** The dependency receives 17; the word is then cleared. A dependency-written replacement value is overwritten by that clear. The next-state global is read afterward.

### Different byte-action observations

**Condition:** Before key 4, \(s=1,c=0,v=0,w=0\). The counter expression observes \(M_c=2,K_c=3\); the pending-state expression observes \(M_v=5,K_v=7\); the address expression observes \(M_a=11\).

**Expected:** The counter becomes 6, the retained intermediate is \(-35\) modulo \(2^{32}\), and the byte destination is \(p+66\) modulo \(2^{32}\). If writable, that destination receives byte value 156. Afterward, the primary working value becomes \(-35\) modulo \(2^{32}\).

If the store changes the multiplier or additive global, it does not change that retained intermediate.

### Default action before address capture

**Condition:** An unlisted key occurs before any word-test action captures an address.

**Expected:** F-5e3309c9d083 receives null. Following normal return, the state becomes the cached fallback.

### Arithmetic destination selection

**Condition:** The arithmetic selector is Boolean zero, with \(T_0=20\) and \(T_1=80\).

**Expected:** The selected state is 23. For selector one, it is 80. This does not require the two occurrences to observe the same value.

### Visible return

**Condition:** Dispatch reaches key 0.

**Expected:** The result’s low 32 bits are zero. No particular upper half or success interpretation is established.

## Uncertainties

The following remain **documented_only**:

- Actual global values, reachable paths, and termination.
- Required storage alignment, writable extent, and allocation relationships.
- Both dependencies’ complete contracts.
- Ownership and lifetime of the input storage and its contained value.
- Character signedness outside Boolean-selector cases.
- Target overflow, address conversion, and right-shift semantics.
- Whether repeated global occurrences can observe different values in the actual execution model.
- Permitted aliasing, external mutation, and concurrency.
- The origin and intended meaning of the return’s upper half.
- Any intended simplification of the arithmetic predicate.

The evidence supports neither a safe-null guarantee nor an unconditional “release and clear” API contract.

## Evidence

**Established static evidence:** The supplied representation contains the ordered entry reads, a global-baseline dispatcher, initially null retained pointers, word tests, two dependency call sites, a post-call clear, occurrence-specific global expressions, a pre-store retained intermediate, calculated byte stores, and a return whose upper-half source has no visible initialization.

**Inferred intent:** The value-based call and clear resemble disposal or handle reset. The counter and byte stores resemble an iterative storage transformation. Neither interpretation determines ownership, safety, or public purpose.

**Unavailable evidence:** Global contents, dependency implementations, callers, allocation metadata, confirmed target ABI, and execution traces.

No runtime tests have been performed.
