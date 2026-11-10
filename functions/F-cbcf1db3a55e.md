<!-- Trusted provenance: F-cbcf1db3a55e; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-cbcf1db3a55e

## Purpose

F-cbcf1db3a55e compares two structured records and returns a signed 32-bit result. Its local decisions establish precedence among flag-defined classes. Depending on the selected class, it compares two-word integers, obtains numeric representations for external comparison, invokes a context-supplied callback, or delegates a prefix-like comparison.

The apparent intent is heterogeneous value ordering. “Numeric,” “integer,” “tag,” and “extent” describe observed operations or inferred roles, not an independently established application-level type system. In particular, flag `0x0001` must not be assumed to mean null, missing, or invalid.

The result is not necessarily −1, 0, or 1. Callback and comparison-dependency results can be returned unchanged; an equal-prefix path returns a wrapping 32-bit subtraction.

## Interface

### Inputs and record layout

The function receives four inputs:

1. **Left record address.**
2. **Right record address.**
3. **Comparison context:** a 32-bit address representation; zero disables the context path.
4. **Auxiliary value:** an opaque 32-bit value supplied twice to F-7f14f1ab6851.

The supplied representation uses 32-bit record words and addresses. Relevant record fields are:

| Byte offset | Width | Observed role |
|---|---:|---|
| `0x00` | 32 bits | Low integer word; first numeric-conversion argument |
| `0x04` | 32 bits | High integer word; second numeric-conversion argument |
| `0x08` | 64 bits | Stored numeric representation |
| `0x14` | 32 bits | Payload-like value for prefix comparison or direct callback |
| `0x18` | 32 bits | Extent-like value for prefix comparison or callback |
| `0x1c` | 32-bit word read, low 16 bits retained | Classification flags |
| `0x1f` | 8 bits | Left-record tag used on the context path |

The tag at `0x1f` is outside the retained low 16 classification bits. It is not another classification flag.

A context, when actually accessed, has these fields:

| Byte offset | Width | Observed role |
|---|---:|---|
| `0x04` | 8 bits | Tag used for comparison and, on mismatch, a later normalization argument |
| `0x08` | 32 bits | Opaque first callback argument |
| `0x0c` | 32 bits | Callback address |

### Output

The output is interpreted as a signed 32-bit integer:

- Classification and locally resolved numeric branches return −1, 0, or 1.
- Context comparison returns the callback result unchanged.
- Prefix comparison propagates a nonzero dependency result unchanged.
- A zero prefix-comparison result leads to a 32-bit extent difference.

Sign-based ordering therefore depends on external contracts on delegated paths. Unrestricted extent subtraction is not guaranteed to have the sign of the mathematical length difference.

### Path-dependent preconditions

Both records must support the initial flag reads. Other fields need to be accessible only when their selected path reads them. No explicit null, bounds, alignment, callback-validity, or extent-validity checks appear.

**A nonzero context is not, by itself, required to be dereferenceable.** This function accesses it only when both captured flags contain `0x0002` and neither contains `0x0001`, `0x0004`, or `0x0008`. On that branch, context fields must be readable when used and the selected callback must be callable. Classification returns, numeric paths, and prefix-comparison paths do not locally dereference the supplied context, even if it is nonzero.

Record writability, payload validity, alignment requirements, and any stronger preconditions imposed by dependencies remain dependency-specific. The absence of a local context dereference does not establish that an arbitrary dependency cannot access related memory through its own state.

## Behavior

### Classification capture and precedence

The function first reads the left classification word at `0x1c`, retains its low 16 bits, and then does the same for the right record. All classification decisions use these captured values. They are not refreshed after dependency calls.

Only four retained bits affect local branch selection:

- `0x0001`: highest-priority class.
- `0x0002`: context-capable class within the nonnumeric branch.
- `0x0004`: integer representation eligible for direct comparison.
- `0x0008`: stored numeric representation eligible for delegated numeric comparison.

Other retained bits have no shown effect.

If either record has `0x0001`, the result is the right bit value minus the left bit value:

| Left `0x0001` | Right `0x0001` | Result |
|---|---|---:|
| Set | Clear | −1 |
| Clear | Set | 1 |
| Set | Set | 0 |

No dependency is called on these paths. Both records having this bit compares equal regardless of all other flags and payloads.

Otherwise, `0x0004` or `0x0008` identifies membership in the numeric branch. A numeric record precedes a record with neither bit. This class distinction returns −1 or 1 without obtaining numeric payloads.

The resulting local precedence is:

1. Records with `0x0001`.
2. Records without `0x0001` but with `0x0004` or `0x0008`.
3. Records with none of those bits.

### Direct integer comparison

When both numeric records carry `0x0004`, the function compares the two-word values at `0x00` and `0x04`.

The low words are compared unsigned to supply a borrow. The high-word comparison incorporates signed-overflow correction. Together these establish signed 64-bit ordering, with `0x04` as the high word and `0x00` as the low word:

- Left smaller: −1.
- Right smaller: 1.
- Equal: 0.

This route takes precedence even if either or both records also carry `0x0008`; the stored representation at `0x08` is not selected.

The reverse ordering check, when needed, reads the integer words again rather than relying on a single atomic record snapshot. There is no intervening dependency call on this route.

### Delegated numeric comparison

When both records are numeric but at least one lacks `0x0004`, the function obtains the left representation before the right representation.

For each record, its captured flags determine the source:

- With `0x0008`, read the 64-bit representation at `0x08`.
- Without `0x0008`, call F-8d7acd353fe6 with the 32-bit words at `0x00` and `0x04`, in that argument order.

The left result is retained while the right is obtained. Thus, a left-side conversion can precede right-side field reads. No assumption is made that conversion preserves the right record or other memory.

Each resulting representation is supplied to comparison dependencies as a low 32-bit word followed by a high 32-bit word. The four arguments represent left low, left high, right low, and right high.

F-11ed8242c40c is called first. If its represented carry condition is clear, this function returns −1 without calling the second comparator. Otherwise, F-c6bfc9d0fb36 receives the same representation pairs. A clear carry condition then produces 1; otherwise the result is zero.

These decisions depend on carry conditions, not established ordinary return-value meanings. The numeric format, predicates, rounding, and treatment of special values are unresolved.

### Nonnumeric selection

When neither record has `0x0001`, `0x0004`, or `0x0008`, bit `0x0002` determines further precedence:

- Left clear, right set: return 1.
- Left set, right clear: return −1.
- Both set and context nonzero: use context comparison.
- Both set and context zero: use prefix comparison.
- Both clear: use prefix comparison regardless of the context value.

A nonzero context does not override these rules.

### Context comparison: matching tag

The function reads and saves the left byte at `0x1f`, then compares it with a byte read from context offset `0x04`. It does not inspect the original right tag.

If those bytes match, no normalization dependency is called. The context callback receives five 32-bit arguments:

1. Context field `0x08`.
2. Left field `0x18`.
3. Left field `0x14`.
4. Right field `0x18`.
5. Right field `0x14`.

The callback result is returned unchanged. This path neither verifies that the right tag matches nor performs subsequent restoration calls.

The table specifies argument positions, not an independently established ordering among all individual loads used to prepare the callback.

### Context comparison: mismatched tag

If the initial left and context tag bytes differ, the saved original left tag is retained through the following sequence:

1. Invoke F-21f3f42d1b03 for the left record. Only the record argument is exposed for this invocation; an additional tag argument is not established.
2. Retain its 32-bit result. If nonzero, read the left field at `0x18`; if zero, select zero without that conditional field read.
3. **Read context byte `0x04` again**, after the first invocation and its left-field selection. Supply this newly read byte as the tag argument when invoking F-21f3f42d1b03 for the right record.
4. Retain the right invocation’s 32-bit result. If nonzero, read right field `0x18`; otherwise select zero.
5. Invoke the context callback with the context field at `0x08`, selected left extent or zero, left dependency result, selected right extent or zero, and right dependency result.
6. Preserve the callback result.
7. Invoke F-21f3f42d1b03 for the left record with the saved original left tag.
8. Invoke F-21f3f42d1b03 for the right record with that same saved tag.
9. Return the preserved callback result.

The second context-tag read is operationally significant. The first normalization invocation may modify context offset `0x04`. The right invocation is therefore **not guaranteed to receive the tag used in the initial comparison**.

The callback address at context offset `0x0c` and its opaque argument at `0x08` are accessed for the callback after both initial normalization invocations. They are not established entry-time snapshots either.

Zero normalization results do not abort comparison. Each zero is passed to the callback and forces its paired extent argument to zero. Later normalization calls still occur if the callback returns normally.

The final two calls suggest restoration intent, but both receive the saved **left** tag. The original right tag is not saved. Their results are ignored; neither successful restoration nor rollback is guaranteed.

### Prefix comparison

The function selects the smaller of the two fields at `0x18` using signed 32-bit ordering. The selected 32-bit pattern is passed unchanged as the extent to F-7f14f1ab6851.

That dependency receives:

1. Left field `0x14`.
2. Right field `0x14`.
3. Selected extent.
4. Auxiliary input.
5. The same auxiliary input again.

A nonzero result is returned unchanged.

If the result is zero, the extent fields are read again after the dependency returns. The function subtracts the right extent from the left extent using 32-bit arithmetic and returns that bit pattern as a signed result.

The subtraction wraps modulo \(2^{32}\). It is not a normalized three-way comparison. Negative signed extents are not rejected before the dependency call, and the final extents need not equal those observed before it.

## Side effects

No direct record or context writes are shown. All potential mutations through called dependencies remain relevant:

- Normalization may modify either record, context, payload storage, or globals.
- The callback may modify memory or state.
- Numeric conversion, numeric comparison, and prefix comparison are not established to preserve memory or globals.
- Captured classification bits remain fixed despite later mutation; later field reads can observe changed data.
- The context tag is explicitly reread between the first and second normalization invocations.
- Prefix-path subtraction uses post-dependency extent reads.
- Final normalization return values do not replace the saved callback result.

No allocation, release, reference-count operation, or ownership transfer is directly established. Passing a record address, payload-like value, or normalization result does not establish that a dependency owns, retains, or releases it.

No local error-recovery protocol is shown. Ordinary zero results have the branch-specific meanings described above; they are not uniformly failures. Faults, nonlocal exits, or other dependency failures have no demonstrated recovery or restoration guarantee.

## Dependencies

| Opaque ID | Established interaction | Unresolved contract |
|---|---|---|
| F-7f14f1ab6851 | Receives two payload-like values, selected extent, and duplicated auxiliary input; nonzero result propagated | Comparison semantics, valid extents, memory accesses, ownership, side effects |
| F-8d7acd353fe6 | Receives two 32-bit words and produces a 64-bit representation | Numeric encoding, rounding, exceptional inputs, mutation |
| F-11ed8242c40c | First comparison of two representation pairs; clear carry produces −1 | Exact predicate, full interface, special-value behavior |
| F-c6bfc9d0fb36 | Second comparison; clear carry produces 1 | Exact predicate, full interface, special-value behavior |
| F-21f3f42d1b03 | Invoked before and after mismatched-tag callback comparison; initial results become callback arguments | First invocation’s complete interface, tag interpretation, mutation, allocation, failure meaning, restoration |

The indirect callback is an additional context-supplied dependency. Its argument positions and result propagation are established; its implementation, ownership rules, and state effects are not.

## Acceptance examples

These are conditional criteria derived from static evidence, not executed tests.

1. **Highest-priority classification.** If only the left record has `0x0001`, return −1. If both have it, return zero regardless of payload differences. No dependency is called.

2. **Unused nonzero context.** If classification selects an early return, this function does not dereference the context. Likewise, two nonnumeric records lacking `0x0002` select prefix comparison without locally accessing context fields.

3. **Numeric precedence.** With neither `0x0001` set, a left record carrying `0x0008` precedes a right record carrying neither numeric bit. Return −1 without numeric conversion.

4. **Integer borrow and sign.** With both `0x0004` bits set, high/low `0x00000000`/`0xffffffff` precedes `0x00000001`/`0x00000000`. An all-ones pair precedes a zero pair. Additional `0x0008` bits do not redirect the path.

5. **Delegated numeric decisions.** Clear carry after F-11ed8242c40c produces −1 and suppresses F-c6bfc9d0fb36. Set carry after both produces zero.

6. **Matching left tag.** With context comparison selected and the initial tags matching, invoke the callback directly. A result of 27 is returned as 27, with no normalization calls, regardless of the uninspected right tag.

7. **Context-tag mutation.** Suppose the initial saved left tag is A and the initial context tag is B, with A differing from B. If the first normalization invocation changes context offset `0x04` to C, and no subsequent change occurs before its reread, the right normalization invocation receives C, not the previously observed B.

8. **Zero normalization result.** On mismatch, a zero left normalization result supplies zero for both left-associated callback arguments and skips the conditional left `0x18` read. Right normalization and callback invocation still proceed.

9. **Final tag arguments and preserved result.** If original left and right tags are A and B, both final normalization calls receive saved A. If the callback returns −17 and both final calls return normally, return −17 regardless of their return values.

10. **Equal prefix.** If F-7f14f1ab6851 returns zero and post-call extents are 5 and 8, return −3.

11. **Signed extent selection.** Extents `0xffffffff` and `0x00000001` select `0xffffffff`. Dependency acceptance of that extent is unresolved.

12. **Subtraction overflow.** If post-call extents are `0x7fffffff` and `0xffffffff`, equal-prefix subtraction produces `0x80000000`, interpreted as −2147483648, not the positive mathematical difference.

## Uncertainties

- **documented_only — First normalization interface.** Only its record argument is exposed. Supplying an assumed tag would exceed the evidence.
- **documented_only — Numeric semantics.** Representation format, carry-based predicates, unordered values, infinities, and signed zero remain unresolved.
- **documented_only — Tag interpretation.** Tags may denote encodings or representations, but that is inferred intent. The full argument-level interpretation of the saved byte is not established.
- **documented_only — Restoration.** Final calls are observable obligations, not proof of successful restoration, especially for the original right state.
- **documented_only — Aliasing and mutation.** No nonaliasing requirement is shown. Sequential calls and delayed reads may make aliasing observable.
- **documented_only — Invalid inputs.** No recoverable behavior is established for inaccessible selected fields, invalid callbacks, unsupported alignment, or invalid extents.
- **documented_only — General ordering guarantees.** Arbitrary dependency behavior and wrapping subtraction prevent an unconditional claim of antisymmetry, transitivity, or total ordering.

## Evidence

Established static evidence comprises the supplied function representation and opaque dependency mapping: access offsets and widths, initial flag capture, branch precedence, signed extent selection, argument positions, dependency sequence, conditional reads, explicit context-tag reread, return propagation, and 32-bit subtraction.

Signed 64-bit ordering on the direct-integer path follows from low-word unsigned borrow and overflow-corrected high-word comparisons. Heterogeneous comparison, numeric conversion, prefix comparison, and restoration are inferred roles where external semantics are unavailable.

No dependency implementations were supplied. No runtime tests have been performed.
