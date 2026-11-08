<!-- Trusted provenance: F-f7e2f4cd400d; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-f7e2f4cd400d

## Purpose

F-f7e2f4cd400d compares an original requested 64-bit position with a two-word position supplied through temporary storage by a dependency. Equality produces a zero return. Otherwise, a context discriminator selects preparation for delegation or an attempt to update a context word directly.

When the direct update is not selected, the function delegates the original request and then invokes a refresh dependency. Refresh occurs even when delegation returns a nonzero status.

“Position,” “cached range,” “cursor,” “preparation,” and “refresh” describe **inferred intent**. The established contract is the argument flow, memory accesses, arithmetic, branch conditions, dependency order, and return behavior below. The resource type and position units are unknown.

## Interface

### Arguments and result

The supplied representation exposes four argument slots:

| Argument | Width | Established role |
|---|---:|---|
| Context address | 32 bits | Identifies directly accessed storage and is passed to dependencies |
| Uninterpreted argument | 32 bits | Not explicitly used by this function |
| Requested low word | 32 bits | Low half of the original requested position |
| Requested high word | 32 bits | High half of the original requested position |

The original request is the unsigned 64-bit quantity formed from its high and low words. Full-position ordering is unsigned: high words decide first; low words decide when high words match.

No role as an origin, option, mode, or handle is established for the unused argument.

The result is a 32-bit status:

- A nonzero result from F-cdf60417d657 is returned immediately.
- A nonzero low result word from F-a78a7790af0e is returned immediately.
- Equality and a completed direct cursor update return zero.
- Delegation returns the saved result from F-7ef43f7b70dc, after F-a042d20fe1ce completes.

No error-code vocabulary or translation is established.

### Context layout

Offsets are relative to the context address. Names describe observed roles rather than proven field semantics.

| Offset | Width | Observed role |
|---|---:|---|
| `+0x05` | 8 bits | Zero/nonzero discriminator |
| `+0x08` | 32 bits | Handle-like word, read separately for delegation and refresh |
| `+0x18` | 32 bits | Cursor-like word, read for a guard and read again for an update |
| `+0x1c` | 32 bits | Base-like word used in range arithmetic |
| `+0x20` | 32 bits | End-like word, read for a guard and again for range arithmetic |
| `+0x28` | 32 bits | Anchor low word, captured once during range construction; also the address passed to refresh |
| `+0x2c` | 32 bits | Anchor high word, read separately for the lower and upper endpoints |

Temporary storage contains two adjacent 32-bit words initialized to the requested low word followed by the requested high word. Its address is supplied to F-cdf60417d657. Word placement is established; byte endianness is not.

### Preconditions and ownership

The context must support the direct accesses and dependency operations reached on the selected path. No explicit checks establish non-nullness, sufficient extent, alignment, or field consistency.

The direct-update path requires writable storage at `+0x18`. F-a042d20fe1ce receives `context + 0x28`; its complete access extent is unknown.

The temporary position storage exists for this invocation. Passing it or the context does not establish ownership transfer, permission to retain an address, or allocation and release rules. Dependency effects are not limited to their explicit arguments.

The function does not establish an atomic snapshot of context fields. External synchronization requirements and the validity of concurrent mutation remain **documented_only**.

## Behavior

### Initial dependency and equality shortcut

The function initializes the temporary two-word position from the original request, then calls F-cdf60417d657 with the context address and temporary-storage address. No direct context-field access precedes this call.

If the dependency returns nonzero, that exact 32-bit status is returned. There are no later direct context accesses or dependency calls by this function. Effects already caused by the dependency are not undone.

If the dependency returns zero, the two temporary words it leaves supply the **comparison position**. Whether this position is current, normalized, logical, or physical is unknown.

If both comparison words equal the original requested words, the function returns zero. It does not read the discriminator or invoke any later dependency.

All remaining behavior requires initial success and unequal positions.

### Discriminator selection

The function reads the byte at `+0x05`.

**Zero discriminator:** F-a78a7790af0e is called with the context address. Its exposed result contains two 32-bit words. The low word is checked as a status; the high word supplies an auxiliary delegation argument.

A nonzero low word is returned immediately. A zero low word proceeds directly to delegation. This branch does not attempt the cached guard, range checks, or direct cursor update.

**Nonzero discriminator:** the function attempts cached handling. If the guard or either range comparison rejects it, the function calls F-36c2c2b132fa with the context address, then proceeds to delegation.

There is no explicit status check after F-36c2c2b132fa. The outgoing secondary 32-bit value exposed after that call is forwarded as delegation’s auxiliary argument. Its formal result-interface status and meaning are unknown.

### Cached guard and endpoint reads

Cached handling first compares the cursor-like word at `+0x18` with the end-like word at `+0x20`, using unsigned 32-bit order.

If cursor is greater than end, cached handling is rejected without constructing endpoints. F-36c2c2b132fa and delegation follow.

If the guard passes, endpoint construction has these established read relationships:

1. The end word at `+0x20` is read again and combined with the base word at `+0x1c` to form the 32-bit difference described below.
2. The anchor low word at `+0x28` is read once and captured.
3. The anchor high word at `+0x2c` is read to calculate the lower endpoint’s high word.
4. The anchor high word at `+0x2c` is read again to calculate the upper endpoint’s high word.
5. The endpoint comparisons follow.

Thus, both high-word reads occur before the lower-bound comparison, including when that comparison will reject the request. The low anchor word is not reloaded for the upper endpoint.

No finer ordering is established between operand loads within a single expression, including the guard operands or the end/base operands used to form the difference.

Define:

- \(D=(\text{end read for arithmetic}-\text{base}+1)\bmod 2^{32}\).
- \(L\): the single captured anchor low word.
- \(H_{\mathrm{lower}}\): the first anchor high-word read.
- \(H_{\mathrm{upper}}\): the subsequent anchor high-word read.
- \(s=0\) if D’s top bit is clear, and \(s=-1\) if it is set.
- \(b=1\) if unsigned \(L<D\), otherwise \(b=0\).

The independently constructed endpoint words are:

| Endpoint | Low word | High word |
|---|---|---|
| Lower | \((L-D)\bmod 2^{32}\) | \((H_{\mathrm{lower}}-s-b)\bmod 2^{32}\) |
| Upper | \((L-1)\bmod 2^{32}\) | \((H_{\mathrm{upper}}-1+\operatorname{indicator}(L\ne0))\bmod 2^{32}\) |

Equivalently, define two potentially different 64-bit values:

\[
A_{\mathrm{lower}}=H_{\mathrm{lower}}2^{32}+L,\qquad
A_{\mathrm{upper}}=H_{\mathrm{upper}}2^{32}+L.
\]

Then:

\[
\text{lower}
=
A_{\mathrm{lower}}
-\operatorname{signextend}_{32\rightarrow64}(D)
\pmod {2^{64}},
\]

\[
\text{upper}
=
A_{\mathrm{upper}}-1
\pmod {2^{64}}.
\]

**Only when the two high-word reads agree** may both endpoints be expressed using one shared anchor \(A\):

\[
\text{lower}=A-\operatorname{signextend}_{32\rightarrow64}(D),
\qquad
\text{upper}=A-1,
\]

with both operations modulo \(2^{64}\).

A shared anchor is therefore a conditional simplification, not the general read contract. Even under that simplification, D is not necessarily a nonnegative 64-bit length: a set top bit contributes a negative sign-extended value.

### Range acceptance

The request must satisfy both inclusive comparisons:

\[
\text{requested}\ge\text{lower},
\qquad
\text{requested}\le\text{upper}.
\]

Both use unsigned 64-bit order. The lower comparison is evaluated first; the upper comparison is reached only if the lower comparison succeeds. Endpoint construction, including the second high-word read, has already occurred.

These comparisons do not describe a circular interval. If the independently constructed lower endpoint is unsigned-greater than the upper endpoint, no request can pass both checks.

The function does not separately require base to be at most end, D to be positive as a signed value, equal high-word reads, or endpoint arithmetic without wraparound.

### Direct cursor update

If the guard and both endpoint comparisons pass, the function reads the cursor at `+0x18` again and writes one 32-bit result back to that offset.

The written value is equivalent to:

\[
\text{new cursor}
=
\text{cursor read for update}
+\text{requested low word}
-\text{comparison low word}
\pmod {2^{32}}.
\]

The operation selects a subtraction form when the full unsigned request is less than or equal to the full comparison position, and an addition form when it is greater. Both forms have the modular result above.

The cursor used for this update is a fresh read, not a captured guard value. No carry or borrow is written to another field. There is no post-update range check, and the cursor result can wrap modulo \(2^{32}\).

After the store, the function returns zero without preparation, delegation, or refresh.

### Delegation and refresh

Delegation is reached after either:

- A zero discriminator and a zero low result word from F-a78a7790af0e; or
- A nonzero discriminator, rejection of cached handling, and completion of F-36c2c2b132fa.

The observable sequence is:

1. Read the handle-like word at `+0x08`.
2. Call F-7ef43f7b70dc with that handle, the auxiliary word, and the original requested low and high words.
3. Save its 32-bit result.
4. Read the handle-like word at `+0x08` again.
5. Call F-a042d20fe1ce with the newly read handle and `context + 0x28`.
6. Return the saved delegation result.

The comparison position does not replace the original request in the delegation arguments.

Refresh occurs for both zero and nonzero delegation statuses. Its result does not replace the saved status. The handle reads are distinct: preparation may affect the first value, and delegation may affect the second.

## Side effects

The only explicit direct context write is the 32-bit cursor store at `+0x18`. The function also initializes temporary position storage before its first dependency call.

Dependencies may have additional memory, resource, global-state, or external effects. In particular:

- F-cdf60417d657 receives the context and writable temporary position storage.
- Both preparation dependencies receive the context before the delegation handle is read.
- Delegation runs before the second handle read.
- Refresh receives an interior context address.

No dependency is assumed to preserve context fields or globals, retain or release memory, or obey an ownership policy not independently established.

There is no explicit allocation, deallocation, ownership transfer, or rollback in this function. Dependency-side ownership effects remain **documented_only**. Initial or preparation failure can leave prior dependency effects in place. Delegation failure still leads to refresh.

The described return sequence assumes dependency calls complete normally. Nonlocal exits and other failure mechanisms are not established.

## Dependencies

| Opaque ID | Established interaction | Unresolved behavior |
|---|---|---|
| F-cdf60417d657 | Receives context and temporary position address; nonzero status stops processing | Position semantics, mutation scope, ownership and external effects |
| F-a78a7790af0e | Used for zero discriminator; low result word controls failure; high word is forwarded | Preparation semantics and auxiliary-word meaning |
| F-36c2c2b132fa | Used after rejected cached handling; outgoing secondary word is forwarded without a status check | Formal result interface, failure reporting, auxiliary meaning |
| F-7ef43f7b70dc | Receives freshly read handle, auxiliary word, and original request; supplies saved return status | Resource semantics, mutations, error meanings |
| F-a042d20fe1ce | Receives separately reloaded handle and `context + 0x28`, regardless of delegation status | Access extent, result meaning, failure and ownership behavior |

## Acceptance examples

These are conditional static expectations, not executed tests. Unless an example explicitly distinguishes reads, numeric field values are assumed stable during all relevant accesses.

1. **Initial failure.**  
   If F-cdf60417d657 returns a nonzero status, return it unchanged. Do not access context fields directly or invoke later dependencies. Retain any effects already produced by that call.

2. **Equality shortcut.**  
   If initial completion succeeds and both temporary words equal the original request, return zero without reading the discriminator.

3. **Zero-discriminator failure.**  
   After initial success with unequal positions, a zero discriminator selects F-a78a7790af0e. If its low result word is nonzero, return that word without cached handling, delegation, or refresh.

4. **Ordinary interior update.**  
   Let the comparison position be 994 and request 997. Let the discriminator be nonzero, cursor 104, base 100, end 109, captured anchor low word 1000, and both anchor high-word reads zero. D is 10, and the inclusive range is 990 through 999. Store cursor 107 and return zero without later calls.

5. **Inclusive endpoints.**  
   With the same fields and comparison position as example 4, request 990 stores cursor 100; request 999 stores cursor 109. Both return zero. Request 1000 exceeds the upper endpoint and instead selects preparation, delegation, and refresh.

6. **Zero discriminator overrides suitable range values.**  
   With the numeric values from example 4 but a zero discriminator, do not attempt a direct cursor update. Call F-a78a7790af0e. Its zero low result word leads to delegation and refresh, not an automatic zero return.

7. **Guard rejection.**  
   After initial success with unequal positions and a nonzero discriminator, an unsigned cursor greater than the guard’s end value rejects cached handling before endpoint construction. Call F-36c2c2b132fa, then delegate and refresh.

8. **Sign-sensitive difference.**  
   Suppose the guard passes, D is `0x80000000`, the captured low word is zero, and both high-word reads are zero. Lower is `0x0000000080000000`; upper is `0xffffffffffffffff`. Treating D as an unsigned 64-bit subtraction operand would produce the wrong lower endpoint.

9. **Different high-word reads.**  
   Suppose the guard passes, D is 10, the captured low word is 1000, the first high-word read is 1, and the second is 2. Lower is `0x00000001000003de`; upper is `0x00000002000003e7`. Request `0x0000000200000384` passes both endpoint checks. If the comparison position is `0x0000000200000383` and the update cursor read is 104, store 105 and return zero. Reusing either high word for both endpoints would incorrectly reject this request. This example is conditional on an execution environment permitting the distinct observed values; it does not establish safe concurrent mutation.

10. **Independently constructed empty interval.**  
    With D 10 and captured low word 1000, a first high-word read of 2 and second read of 1 produce lower `0x00000002000003de` and upper `0x00000001000003e7`. No request passes. Both high-word reads occur before rejection by the endpoint comparisons.

11. **Delegation failure and handle mutation.**  
    If delegation returns nonzero and changes the handle field, refresh receives the changed handle. After refresh completes, return the saved nonzero delegation status, not the refresh result.

## Uncertainties

The following remain **documented_only**:

- Resource type, position units, and semantic meanings of context fields.
- Meaning of the comparison position supplied through temporary storage.
- Formal interface and meaning of the secondary word after F-36c2c2b132fa.
- Dependency mutation scope, global-state behavior, ownership, retention, allocation, and release.
- Refresh access extent, result semantics, and failure mechanisms.
- Error definitions and non-return-based failures.
- Byte endianness, alignment requirements, synchronization rules, and external field invariants.
- Whether differing high-word observations can arise in a valid execution environment.
- Intended handling of sign-sensitive D values, wrapped endpoints, and wrapped cursor updates.

These unknowns do not authorize added validation, a synthesized coherent anchor snapshot, circular-interval semantics, rollback, or ownership guarantees.

## Evidence

**Established static evidence:** the supplied representation exposes argument widths, context offsets, temporary-word placement, status propagation, unsigned comparisons, modular arithmetic, the direct cursor store, dependency ordering, separately reloaded handles, and refresh after either delegation status.

For endpoint construction specifically, it shows one captured low anchor word followed by two distinct high anchor reads: the lower high word is calculated from the first, and the upper high word from the second. Both calculations precede the range comparisons. The general formulas above preserve those independent observations.

**Inferred intent:** positioning, cached-range, cursor, preparation, and refresh terminology describes apparent roles without establishing dependency semantics.

**Evidence limits:** the supplied representation carries a type-propagation warning. Recovered type labels do not establish a stronger source-language type model or dependency ABI than the visible operations support.

No runtime tests have been performed. All acceptance examples are conditional consequences of static analysis.
