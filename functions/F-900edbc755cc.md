<!-- Trusted provenance: F-900edbc755cc; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-900edbc755cc

## Purpose

F-900edbc755cc processes a term list against a separate reference-expression list. Its mode selects whether diagnostics identify the terms as belonging to a GROUP BY or ORDER BY clause.

For each processed term, a dependency returns either a negative failure result, zero, or a positive one-based reference position. A positive position causes the term to be replaced with the result of a duplication dependency applied to the selected reference expression. Under a specific flag condition, one field from the original term is transferred to the replacement.

**Established behavior:** term-count checking, sequential dependency calls, positional bounds checking, conditional replacement, conditional field transfer, and a four-byte status result.

**Inferred intent:** resolve clause terms against a result-expression list while retaining an explicitly marked property of the original term.

This is a static behavioral contract. It does not establish parser semantics, memory ownership, or the complete behavior of any dependency.

## Interface

### Inputs

The function accepts five arguments:

1. **Context handle.** A pointer to a cell containing a context address. The function reads this cell immediately and retains that address for its subsequent direct context accesses and duplication calls.
2. **Reference-list handle.** A pointer to a cell containing a reference-list address. This handle is also passed to the term-resolution dependency.
3. **Term-list address.** A nullable address identifying the terms to process.
4. **Clause mode.** A four-byte integer. Zero selects GROUP BY diagnostic wording; every nonzero value selects ORDER BY wording. No other direct behavior depends on this mode.
5. **Forwarded value.** An opaque four-byte value passed unchanged to each term-resolution call. This function does not otherwise interpret it.

### Observed layouts

The supplied representation uses four-byte integers and address-bearing fields, two-byte flags, and one-byte state. The following offsets are byte offsets. No fields outside those listed are established.

| Object | Offset | Observed meaning |
|---|---:|---|
| Context | 0x1e | One-byte processing-bypass state |
| Context | 0x58 | Signed four-byte maximum term count |
| Either list object | 0x00 | Signed four-byte entry count |
| Either list object | 0x0c | Address of an entry array |
| Entry array | Each 0x0c-byte record, offset 0x00 | Expression address |
| Expression | 0x02 | Unsigned two-byte flags |
| Expression | 0x04 | Four-byte field eligible for transfer |

The remaining eight bytes of each list-entry record are not directly read or written here.

### Output

The function returns a four-byte status:

- **1:** an explicitly detected error.
- **0:** completion, bypass, absence of a reference list, or another path that reaches the normal return.

A zero result does **not** establish that all term expressions are nonnull, that duplication succeeded, or that dependencies reported no error through other state.

### Preconditions

The context handle must be readable even when the term-list address is null: its contained context address is fetched before the null-term-list check.

When the term-list address is nonnull, the retained context address must support the bypass-byte read. Further readable or writable storage is required only as the selected path reaches it.

For ordinary processing, the caller must provide valid list headers, entry arrays, expression objects, and writable term-entry expression fields for every accessed index. The function performs no general pointer validation.

Aliasing, concurrent mutation, and dependency-induced invalidation are not protected against. In particular, a previously retained list or expression address is not automatically refreshed merely because a dependency receives a related handle.

## Behavior

### 1. Capture context and decide whether to process

The function first reads the context address from the context handle.

If the term-list address is null, it returns 0. It does not read the bypass byte, reference-list handle, list counts, or entry arrays on this path.

Otherwise, it reads the byte at context offset 0x1e. A nonzero byte causes an immediate return of 0. No term-count check or dependency call is made on that path.

The context address is retained from the initial read. Later changes to the context-handle cell do not make the function substitute a new context address for its own duplication calls.

### 2. Enforce the term-count limit

With a nonnull term list and a zero bypass byte, the function compares the context maximum at offset 0x58 with the term-list count at offset 0x00.

Both operands are treated as signed four-byte integers. If the maximum is less than the term count, the function calls F-2022b0f73f3b with:

- The original context handle.
- A diagnostic identifying an excessive number of clause terms.
- GROUP or ORDER wording selected by the clause mode.

It then returns 1.

This check precedes reading the reference-list address. Consequently, a null reference list does not suppress an excessive-term-count error.

There is no separate validation that either count is nonnegative. A negative maximum can therefore reject an otherwise empty term list.

### 3. Capture the reference-list address

After passing the count-limit check, the function reads the reference-list address from the reference-list handle.

If that address is null, it returns 0 without resolving or replacing any term.

Otherwise, that reference-list object address is retained for subsequent count and entry-array accesses. A dependency may change the handle cell, but this function continues to use the retained object address for its direct reference-list accesses.

### 4. Visit terms in increasing index order

Processing starts at zero-based term index zero.

The loop condition reads the term-list count again for each iteration. It is not a fixed snapshot of the count used for the initial maximum check. A dependency-induced count change can therefore shorten or extend processing without another maximum-count comparison.

For each visited term:

1. The function reads the term-list entry-array address.
2. It reads the expression address from the current 12-byte entry.
3. It calls F-343f48b6323a with the context handle, reference-list handle, captured term expression, one-based term number, a literal zero argument, and the forwarded four-byte value.

The captured expression address remains the address used by the subsequent replacement path. The function does not reload that address from the term entry after resolution.

### 5. Interpret the resolution result

The result from F-343f48b6323a is interpreted as a signed four-byte integer.

**Negative result:** return 1 immediately. This function makes no additional diagnostic call on this path.

**Zero result:** make no direct replacement for this term and proceed to the next iteration. This does not imply that the dependency left memory or external state unchanged.

**Positive result:** treat it as a one-based reference position, subject to an upper-bound check.

Before the positive-result handling, the function compares the returned value with the current signed count in the retained reference-list object. If the returned value is greater than that count, it calls F-2022b0f73f3b with the current one-based term number, GROUP or ORDER wording, and the reference count, then returns 1.

This comparison also applies to a zero resolution result. Thus, if the reference count is negative, zero exceeds that count and takes the out-of-range error path. A negative resolution result has already returned and does not reach this comparison.

### 6. Replace a positively resolved term

For an in-range positive position, the function performs the following ordered actions:

1. Reads the original captured expression’s two-byte flags at offset 0x02.
2. Reads its four-byte field at offset 0x04.
3. Calls F-841dc33ee157 with the captured original expression address.
4. Reads the entry-array address from the retained reference-list object.
5. Reads the expression address at the selected one-based reference position.
6. Calls F-1a5398bcc6b3 with the retained context address and the selected reference expression.
7. Reads the term-list entry-array address and writes the returned replacement address into the current term entry.
8. Conditionally updates the replacement expression as described below.

The reference expression is selected using a 12-byte stride: position one selects byte offset zero, position two selects byte offset 12, and so forth.

The original-expression dependency runs **before** the selected reference expression is fetched and before duplication. This order matters if the lists alias, the expressions alias, or that dependency mutates shared state. The function supplies no protection against those interactions.

### 7. Conditionally transfer the saved field

The saved original field is transferred only when all three conditions hold:

- The returned replacement address is nonzero.
- The saved four-byte field is nonzero.
- The saved flags contain bit 0x0100.

When those conditions hold, the function first writes the saved field to replacement offset 0x04. It then reads the replacement’s two-byte flags and writes them back with bit 0x0100 set.

Other replacement flag bits are retained by this direct update. Other original flag bits are not copied by this function.

If duplication returns zero, zero is still written into the term entry. The field-transfer step is skipped, and processing can continue. There is no explicit duplication-result failure return.

### 8. Finish without rollback

After a completed iteration, the function advances to the next term. Once the loop condition fails, it returns 0.

An error on a later term does not reverse earlier replacements, dependency effects, or field transfers. The current term may also have been changed by the resolution dependency before a negative result or range error is returned.

## Side effects

### Direct writes

The function directly writes only:

- The expression-address field of a term entry after a positive in-range resolution.
- Replacement-expression offset 0x04 when the transfer condition holds.
- Replacement-expression flags at offset 0x02 on that same transfer path.

It does not directly update either list count, either list’s entry-array address, the bypass byte, or the configured maximum.

### Dependency-visible effects

Dependencies receive context handles, expression addresses, and other state that may permit additional mutation. Their effects are not bounded by the direct-write list above.

Diagnostic emission is requested on two explicit paths: excessive term count and a resolution value exceeding the reference count. The visible function does not inspect the diagnostic dependency’s result.

### Ownership and lifetime

Calling F-841dc33ee157 before replacement suggests disposal or release intent, but its actual ownership behavior is **documented_only**. This contract does not assert that it frees the expression or preserves related objects.

Likewise, F-1a5398bcc6b3 is used as a duplication operation, but independent allocation, ownership transfer, sharing, and failure reporting are **documented_only**.

No assumption is made that any dependency preserves globals, retained objects, list counts, array addresses, or unrelated memory.

## Dependencies

| Opaque ID | Role visible to this function | Unestablished behavior |
|---|---|---|
| F-343f48b6323a | Resolves a term; returns a signed result interpreted as failure, no replacement, or a one-based reference position | Matching rules, diagnostics, mutation, ownership effects |
| F-841dc33ee157 | Receives the original expression before replacement is obtained | Destruction, release, recursive effects, alias safety |
| F-1a5398bcc6b3 | Receives the retained context address and selected reference expression; returns the replacement address | Allocation, copying depth, null-result meaning, ownership |
| F-2022b0f73f3b | Receives diagnostic formatting inputs on explicit count/range errors | Output destination, context mutations, allocation, error-state changes |

These are call-boundary descriptions, not inferred implementations of the dependencies.

## Acceptance examples

The examples below are conditional static expectations, not executed tests. Unless noted, storage is valid, the bypass byte is zero, arithmetic is representable, and dependencies do not invalidate accessed objects.

1. **Absent term list.**  
   Given a readable context-handle cell and a null term-list address, return 0. Do not read the bypass byte or reference-list handle.

2. **Bypassed processing.**  
   Given a nonnull term list and a nonzero bypass byte, return 0 without reading the term count or invoking dependencies.

3. **Too many terms despite no reference list.**  
   Given maximum 2, term count 3, and a reference handle containing null, invoke the excessive-term diagnostic and return 1. Mode zero selects GROUP wording; mode −1 selects ORDER wording.

4. **No reference list after a valid count check.**  
   Given maximum 4, term count 2, and a null reference-list address, return 0 without term resolution.

5. **Zero resolution with an ordinary reference count.**  
   Given reference count 3 and resolution result zero, perform no direct replacement for that term and continue.

6. **Negative resolution.**  
   Given a resolution result of −1, return 1 without invoking this function’s range diagnostic or replacement dependencies.

7. **Positive out-of-range position.**  
   Given reference count 3 and resolution result 4 for the second term, diagnose an out-of-range second term using the selected clause wording and return 1. Earlier completed replacements remain.

8. **Replacement with field transfer.**  
   Given resolution result 2, reference count at least 2, saved flags containing 0x0100, and a nonzero saved field, call the original-expression dependency before obtaining a duplicate of reference entry two. If the duplicate is nonnull, install it, copy the saved field to offset 0x04, and set its 0x0100 flag while retaining its other flag bits.

9. **Replacement without field transfer.**  
   If either the saved field is zero or the saved flags lack 0x0100, install a nonnull duplicate without directly changing its field or flags.

10. **Null duplication result.**  
    Given an otherwise valid positive resolution and a zero duplication result, write zero to the term entry, skip field transfer, and continue. If no later explicit error occurs, return 0.

11. **Negative reference count.**  
    Given a visited term, reference count −1, and resolution result zero, take the out-of-range diagnostic path and return 1.

12. **Count mutation during resolution.**  
    If a dependency reduces the term count below the next index, stop at the next loop-condition check. If it increases the count, additional entries can be visited without repeating the initial maximum check.

## Uncertainties

- **documented_only — arithmetic extremes.** Counts, positions, loop indices, and comparisons use four-byte signed integers in the supplied representation. Entry addressing uses a 12-byte stride. Behavior on signed overflow, unrepresentable one-based numbering, address wraparound, or invalid resulting addresses is not established. No explicit overflow checks are visible.
- **documented_only — complete object layouts.** Only the accessed offsets and record stride are established.
- **documented_only — dependency contracts.** No preservation, allocation, release, diagnostic, or ownership guarantees are available beyond the visible call boundaries.
- **documented_only — aliasing safety.** The original expression may alias a selected reference expression or shared substructure. Safety depends on unavailable dependency behavior.
- **documented_only — diagnostic details.** The supplied data establishes excessive-term and out-of-range diagnostic intent and argument selection, but not the complete rendered out-of-range message.
- **Inferred intent, not established semantics.** The lists likely represent clause expressions and reference/result expressions. The exact language rules and meaning of the transferred field are unknown.

## Evidence

This contract is derived solely from the supplied static function body and dependency map. It records visible conditions, field widths, offsets, calls, return paths, and ordered reads and writes where that order is established.

No implementation files, dependency bodies, prior documents, or external artifacts were consulted. No runtime tests were performed. The acceptance examples are conditional consequences of the visible control flow, not observed execution results.
