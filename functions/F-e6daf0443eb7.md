<!-- Trusted provenance: F-e6daf0443eb7; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-e6daf0443eb7

## Purpose

F-e6daf0443eb7 traverses two address-like ranges backward in **12-byte steps**, uses a dependency to choose between their current positions, and invokes a second dependency after decrementing an output cursor. Once one range is exhausted, it continues a final traversal over the other range and returns the resulting output cursor.

The visible structure suggests a backward merge of fixed-size records. That is **inferred intent**, not an established data-processing guarantee: the ordering relation, record representation, transfer semantics, and arguments to the final-traversal dependency calls are unresolved.

The timing of lower-bound reads is significant. The first range’s lower boundary is saved before any dependency call, but left-exhaustion checks later read the first input again. Which lower boundary governs final traversal depends on which range is exhausted.

This contract describes static evidence only. No runtime tests have been performed.

## Interface

The supplied representation declares five integer-valued inputs and an integer-valued result. The following names identify their behavioral roles, not independently established types:

| Position | Name | Observed role |
|---|---|---|
| First | L₀ | First range’s lower-bound input; read initially and during left-exhaustion checks |
| Second | L₁ | First range’s initial upper boundary; subsequently used as a backward candidate cursor |
| Third | R₀ | Second range’s lower-bound input; read initially, during right-exhaustion checks, and on one transition to final traversal |
| Fourth | R₁ | Second range’s initial upper boundary; subsequently used as a backward candidate cursor |
| Fifth | D | Output cursor, decremented before each transfer-like dependency call |

For ordinary range use, the intervals are interpreted as [L₀, L₁) and [R₀, R₁). This interpretation follows from subtracting 12 from each nonempty range’s upper boundary before selecting its first candidate.

The representation uses signed integer declarations and address-like operations. It does **not** independently establish the integer width, address width, calling convention, or overflow behavior. In particular, a 12-byte stride does not establish a 32-bit address model or a three-field record layout.

The result is the final output-cursor value. No separate status, count, allocation, or constructed object is returned.

### Preconditions for ordinary finite traversal

The function performs no explicit validation. Its apparent intended use requires:

- Each range’s size to be nonnegative and divisible by 12.
- All required arithmetic to be representable under the actual execution model.
- Every dependency invocation to receive values valid under that dependency’s contract.
- Enough usable destination positions below the incoming D, if the transfer-like dependency actually writes there.
- Dependencies to return normally and preserve traversal-relevant state except where a separately established contract permits mutation.

These are external conditions, not enforced checks. Equality alone determines emptiness and exhaustion. There are no visible ordering, alignment, divisibility, capacity, or overlap checks.

## Behavior

### Initial selection and saved boundaries

Before checking whether the first range is empty, the function initializes final-traversal state from the second range: its upper boundary is assigned first, followed by its lower boundary.

It then compares L₀ with L₁.

- **If equal:** the second-range final-traversal state remains selected. The function skips the two-range selection phase.
- **If unequal:** the function replaces the final-traversal state with the first range’s upper and lower boundaries, in that order. Only then does it test whether R₀ equals R₁.

Consequently, when the first range is initially nonempty, its lower boundary is saved:

1. After the first-range nonemptiness check.
2. Before the second-range emptiness check.
3. Before any dependency call.

Call this retained value the **saved left lower boundary**. It is distinct from a later read of the first input’s storage.

If the second range is empty, final traversal uses the saved first-range bounds. If both ranges are nonempty, selection begins, and the saved left lower boundary remains available for the transition caused by right-range exhaustion.

If both ranges are empty, no dependency is called and D is returned unchanged.

These observations describe the ordering of visible state assignments and conditions. They do not establish physical instruction-level load ordering.

### Initial candidates

When both ranges are nonempty, the function decreases the first range’s upper cursor by 12 and then decreases the second range’s upper cursor by 12.

The initial candidates are therefore:

- Left: L₁ − 12.
- Right: R₁ − 12.

No record-content read is directly visible in this function.

### Selection and output ordering

Each selection decision invokes F-e46aba7df1fa with three explicit arguments:

1. An address into the current invocation’s stack argument area, apparently corresponding to the first incoming argument slot.
2. The current right candidate.
3. The current left candidate.

After this dependency returns, the function decreases the output cursor by 12. It then invokes F-9c1c19a88b73 on the selected branch.

The established sequence is:

1. Selection dependency returns a result.
2. Output cursor decreases by 12.
3. Selected-position dependency is invoked.
4. After that dependency returns, the selected range’s exhaustion condition is evaluated.

Only zero versus nonzero matters. Negative and positive nonzero results take the same branch.

The explicit argument sequence establishes logical argument positions, not an ABI-specific sequence of argument loads or stack writes.

### Nonzero result: select the left candidate

A nonzero selection result causes F-9c1c19a88b73 to receive:

1. The already-decremented output cursor.
2. The current left candidate.

After that call returns, the function compares the **then-read first input** with the current left candidate. It does not use the saved left lower boundary for this check.

If they differ, the left candidate decreases by 12 and selection repeats. The function does not itself advance the right candidate on this path.

If they are equal, selection ends. The function configures final traversal as follows:

1. Its upper cursor becomes the current right candidate plus 12.
2. Its lower boundary is read from the third input at this point.

Thus, this transition uses a **post-call read of the right lower-bound input**, not the second-range lower boundary saved before the initial emptiness checks.

The first input’s original value cannot be substituted for its post-call value without an additional preservation assumption. The selection dependency receives an apparent address of that input’s storage, and neither dependency’s mutation or address-retention behavior is established.

### Zero result: select the right candidate

A zero selection result causes F-9c1c19a88b73 to receive:

1. The already-decremented output cursor.
2. The current right candidate.

After that call returns, the function compares the third input with the current right candidate.

If they differ, the right candidate decreases by 12 and selection repeats. The function does not itself advance the left candidate on this path.

If they are equal, selection ends. The final-traversal upper cursor becomes the current left candidate plus 12. Its lower boundary remains the **saved left lower boundary captured before the second-range emptiness check**.

There is no fresh read of the first input to establish this final-traversal lower boundary. A later change to the first input therefore does not, by itself, replace the retained boundary on this transition.

### Final-traversal boundary selection

The established boundary sources are:

| Entry into final traversal | Lower boundary | Initial upper cursor |
|---|---|---|
| First range initially empty | Initially saved R₀ | Initially saved R₁ |
| First range nonempty, second initially empty | Saved L₀, captured before checking second-range emptiness | Saved L₁ |
| Left candidate reaches the then-read first input | R₀ read after the selected-position call and successful left-exhaustion check | Current right candidate + 12 |
| Right candidate reaches the then-read third input | Saved L₀, captured before any dependency call | Current left candidate + 12 |

The function itself does not refresh the final-traversal lower boundary during the final loop.

### Final traversal

While the final-traversal upper cursor differs from its selected lower boundary, the function:

- Decreases the output cursor by 12.
- Invokes F-9c1c19a88b73.
- After that call returns, decreases the final-traversal upper cursor by 12.

The supplied representation shows **no explicit arguments** for this dependency call site. It therefore does not establish the source, destination, or other values received by the dependency during final traversal. This absence must not be converted into either a verified zero-argument signature or a verified two-argument transfer.

The final upper cursor is decreased after the dependency returns, unlike the candidate initialization preceding selection.

Once the final bounds compare equal, the function returns the current output cursor.

### Conditional counts

Assume initial range sizes of 12m and 12n, where m and n are nonnegative integers. Also assume representable arithmetic and that **every dependency call returns normally and preserves all traversal-relevant state**, apart from the updates explicitly performed by this function.

Under those conditions:

- F-9c1c19a88b73 is invoked exactly m + n times.
- The result is D − 12(m + n).
- F-e46aba7df1fa is never called if either range is initially empty.
- If both ranges are nonempty, selection ends when one range’s selected candidate reaches its lower boundary.

These are control-flow and cursor properties. They do not establish that m + n records are transferred.

## Side effects

### Directly visible effects

The function changes its cursor and saved traversal state, exposes a stack-area address to F-e46aba7df1fa, and invokes the dependencies.

No direct record-content access, allocation, release, or global-state write is visible outside those calls.

### Dependency-mediated effects and ownership

The supplied evidence does not establish whether either dependency:

- Reads or writes records, input storage, destination storage, or globals.
- Copies, moves, swaps, constructs, destroys, allocates, or releases objects.
- Retains a supplied address.
- Supports overlapping input and output regions.
- Reenters this function or alters state consulted after returning.

In particular, F-9c1c19a88b73 cannot be assumed to preserve the first input merely because it is not explicitly passed the exposed stack address. Another dependency could have retained that address, and shared-state behavior is unknown.

Passing a candidate address establishes no ownership transfer. The stack-area address is invocation-associated storage, not an independently established owned object. Any permissible retention and use require a dependency contract absent here.

### Errors and partial progress

There is no visible validation error, status result, rollback, or recovery path.

If a dependency fails to return normally, subsequent calls and the ordinary result are not established. Earlier side effects may remain. In selection, the output cursor has already been decreased before the selected-position dependency runs, but the exhaustion check has not yet occurred.

Malformed bounds and arithmetic overflow have no established failure mode. The evidence does not justify guaranteeing rejection, wraparound, trapping, or nontermination.

## Dependencies

### F-e46aba7df1fa

**Established use:** Three explicit arguments: stack-area address, right candidate, left candidate.

**Established result interpretation:** Zero selects the right candidate; any nonzero value selects the left candidate.

**Documented_only:** Exact signature and widths, ordering relation, tie semantics, memory accesses, mutation rights, address retention, global effects, ownership effects, and abnormal control flow.

A pure-comparator contract is not established.

### F-9c1c19a88b73

**Established selection-phase use:** Two explicit arguments: decremented output cursor and selected candidate.

**Established final-traversal use:** Invocation with unresolved argument values in the supplied representation.

**Established result use:** No return value is inspected.

**Documented_only:** Exact signature, final-traversal arguments, transfer semantics, object lifetime effects, aliasing support, allocation and release behavior, shared-state effects, and failure behavior.

A copy or move contract is not established.

## Acceptance examples

These are conditional static expectations, not executed tests. Except for the no-call example, every example asserting exact call counts or results assumes representable arithmetic and that **all dependency calls return normally and preserve all traversal-relevant state throughout the invocation**.

### Both ranges empty

With L₀ = L₁ and R₀ = R₁:

- Neither dependency is called.
- The result is the incoming D.

No dependency-preservation assumption is needed because there are no dependency calls.

### Only the left range contains two units

With L₁ = L₀ + 24 and R₀ = R₁:

- First-range bounds are saved before second-range emptiness is checked.
- F-e46aba7df1fa is not called.
- Two final-traversal calls to F-9c1c19a88b73 occur.
- The result is D − 24.

The arguments and record effects of those two calls remain unresolved.

### Only the right range contains one unit

With L₀ = L₁ and R₁ = R₀ + 12:

- Initially saved right-range bounds govern final traversal.
- No selection call occurs.
- One final-traversal call occurs.
- The result is D − 12.

### One unit per range; nonzero selection

With L₁ = L₀ + 12 and R₁ = R₀ + 12, and a nonzero first selection result:

- Selection receives the stack-area address, R₀, and L₀.
- The selected-position call receives D − 12 and L₀.
- After that call returns, the first input is read for left exhaustion.
- Under the stated preservation assumption, exhaustion succeeds.
- The third input is then read to configure right-range final traversal.
- One final-traversal call follows, and the result is D − 24.

Preservation by the selection dependency alone is insufficient for this conclusion.

### One unit per range; zero selection

Under the same bounds, with a zero first selection result:

- The selected-position call receives D − 12 and R₀.
- Right exhaustion is tested after that call returns.
- Under the stated preservation assumption, exhaustion succeeds.
- Final traversal uses the previously saved L₀, not a fresh read of the first input.
- One final-traversal call follows, and the result is D − 24.

### Changed first input before left-exhaustion testing

Suppose the initial left range contains one unit and the first selection result is nonzero. If dependency activity changes the first input so its post-call value differs from the current left candidate, left exhaustion does **not** succeed at that point.

The function decreases the left candidate and continues selection. The initial one-unit range size no longer guarantees two total transfer-like calls or result D − 24. No safe eventual outcome is established.

### Saved lower boundary after right exhaustion

Suppose both ranges are initially nonempty. If a zero-result branch reaches right exhaustion, and dependency activity has changed the first input but left the saved boundary and other relevant traversal state intact, final traversal still uses the first range’s lower boundary captured before any dependency call.

The changed first-input value is not reread to configure that boundary.

### Other edge conditions

A negative nonzero selection result selects the left candidate just as a positive nonzero result does.

Bounds separated by a nonmultiple of 12 receive no rounding or rejection. No finite, safe completion guarantee follows.

## Uncertainties

The following remain **documented_only**:

- Integer and address widths, ABI, and overflow semantics.
- Exact identity and accessibility of the exposed stack-area storage.
- Dependency mutation, retention, global-state, ownership, and failure contracts.
- Arguments received during final traversal.
- Record size beyond the observed traversal stride, field layout, alignment, and lifetime rules.
- Input sortedness, comparison relation, tie policy, and stability.
- Supported overlap and destination capacity requirements.

The saved-versus-fresh lower-bound distinctions are established by the visible control flow. Whether dependencies actually exploit the exposed storage is unknown. Neither that uncertainty nor the apparent merge intent justifies treating all lower-bound reads as immutable initial values.

## Evidence

This contract is based solely on the supplied static function representation and opaque dependency mapping.

Established evidence includes:

- Equality-based emptiness and exhaustion checks.
- Saving first-range bounds before checking second-range emptiness.
- Twelve-byte cursor changes.
- Right-before-left candidate arguments.
- Zero/nonzero selection.
- Selected-position calls preceding exhaustion checks.
- Fresh first-input reads for left exhaustion.
- Fresh third-input reads when configuring final traversal after left exhaustion.
- Retention of the saved first-range lower boundary after right exhaustion.
- Unresolved final-traversal call arguments.
- Return of the final output cursor.

No dependency implementations, machine instructions, ABI documentation, caller contracts, record definitions, or runtime observations were supplied. No runtime tests have been performed.
