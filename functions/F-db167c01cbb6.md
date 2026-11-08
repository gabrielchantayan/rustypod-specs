<!-- Trusted provenance: F-db167c01cbb6; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-db167c01cbb6

## Purpose

F-db167c01cbb6 selects a 32-bit word from a context-associated table. Selection depends on a reference, selector, context mode, input flags, and reversal control.

The function can additionally request a transition, invoke a notification dependency, request table initialization, or clear a context flag. These effects belong to distinct branches; they are not a common failure handler.

**Established static behavior:** every normal return is either zero or a 32-bit word loaded from table storage. A zero result therefore does not distinguish rejection, unavailable storage, and successful selection of a zero-valued entry.

**Inferred intent:** the reference identifies a current position, the mode controls advancement or wrapping, and the selector identifies an associated table. These labels describe the visible use of values, not established meanings of the dependencies or stored entries.

## Interface

### Inputs and result

| Input | Represented width | Visible role |
|---|---:|---|
| Context address | 32 bits | Base address for fields, selector-specific table slots, and dependency arguments. |
| Reference | 32 bits | Zero starts selection at position zero; nonzero invokes reference resolution. |
| Selector | 32 bits | Passed unchanged to position calculation, then copied into mutable local storage before table selection. |
| Reverse control | 8 bits | Any nonzero value enables position reversal. |
| Input flags | 8 bits | Bits `0x01`, `0x02`, and `0x04` affect visible behavior. |

Other input-flag bits have no direct effect in the visible body, including when the flags participate in the exhausted-selection condition.

The result is a 32-bit word. Its interpretation as a pointer, handle, scalar, or other encoding is **documented_only**.

### Memory layout

All offsets below are byte offsets.

| Location | Access width | Meaning established by use |
|---|---:|---|
| Context + `0x40` | 32 bits | Descriptor address. |
| Descriptor + `0x2e` | 16 bits, unsigned | Entry count, zero-extended for comparison and reversal. |
| Context + `0x184` | 16 bits, signed | Mode; only zero, one, and two receive supported mode-specific behavior. |
| Context + `0x18c` | 8 bits | Context flags. |
| Context + `0x3ac` + four times mapped selector | 32 bits | Selected table address. |
| Table + `0x10` + four times final position | 32 bits | Selected result word. |

The table entry area begins after a 16-byte prefix. The prefix’s contents and purpose are unknown.

### Preconditions and arithmetic

The caller must provide storage valid for every access reached by the chosen branch. The function does not validate descriptor addresses, mapped-selector bounds, table extent, or entry readability.

A zero context is not a supported general failure input. Count acquisition explicitly substitutes zero when the context is zero, but other accesses lack this protection. For example, after initial eligibility succeeds, a zero reference and zero context lead to the exhausted-selection flag read at context + `0x18c`.

Positions and position comparisons are unsigned 32-bit. Position increments wrap from `0xffffffff` to zero. Counts range from zero through `65535`. Reversal uses unsigned 32-bit subtraction, but the preceding range check ensures a mathematical result between zero and the captured count minus one.

Selector-based addressing uses the mapped 32-bit representation and a four-byte stride. No bounds check or widened, unbounded indexing is established. Valid negative selector values, address-overflow behavior, alignment requirements, and allocated slot bounds remain **documented_only**.

## Behavior

### Initial eligibility

The first operation is a call to F-3b037d560480 without arguments.

If it returns zero, F-db167c01cbb6 immediately returns zero. Before that return, the visible body performs no context-field access and calls no other dependency.

A nonzero result permits candidate construction. It does not replace the later eligibility call.

### Candidate construction and mode selection

The candidate begins at unsigned position zero.

With a zero reference, reference resolution, position calculation, context mode handling, and mode-one wrapping are all skipped. Processing proceeds directly to the first final-range check.

With a nonzero reference:

1. F-6ef0016a253f receives the context and reference.
2. A zero result causes an immediate zero return.
3. Otherwise, F-6fa4e01d8d29 receives that result and the original selector.
4. Its result becomes the unsigned 32-bit candidate.

After position calculation, the context flag byte is read.

If context bit `0x08` is set, the candidate advances once. The mode is not read, and neither mode-one wrapping nor mode-two conditional advancement occurs.

If context bit `0x08` is clear, the signed 16-bit mode is read:

- **Zero:** advance once.
- **One:** apply the mode-one behavior below.
- **Two:** advance once only when input bit `0x04` is set; otherwise retain the calculated position.
- **Any other value:** return zero immediately.

Thus negative modes belong to the unsupported group when mode handling is reached. An unsupported stored mode has no effect when the reference is zero or context bit `0x08` bypasses mode handling.

Every advancement is an unsigned 32-bit increment, including wraparound at the maximum value.

### Mode-one advancement and wrapping

Mode one first advances the candidate once.

When input bit `0x02` is set, no wrap-stage count is read. The incremented candidate proceeds unchanged to the first final-range check.

When input bit `0x02` is clear, a wrap-stage count is obtained. A zero context yields count zero without reading a descriptor; otherwise the descriptor address is read from the context, followed by its unsigned 16-bit count.

If the incremented candidate is below this count, it remains unchanged.

If the candidate is equal to or above the count:

- The candidate is reset to zero.
- The context flag byte is read to examine bit `0x01`.
- If context bit `0x01` and input bit `0x01` are both set, F-8a6c49c858cd receives the context and a literal zero.
- If that call returns nonzero, F-df1f4510064d receives the context.

The reset occurs regardless of those flags or dependency results. It is a reset to zero, not a remainder operation.

Processing continues after the callbacks. Neither callback result directly supplies this function’s return value. Later range checks obtain new counts; the wrap-stage count is not treated as persistent state.

### First final-range check

The function obtains a count again, using the same zero-context rule and descriptor layout.

If the candidate is below this count, processing continues to the second eligibility gate.

Otherwise, the exhausted-selection path:

1. Reads the context flag byte.
2. Checks whether context bit `0x01` and input bit `0x01` are both set.
3. If both are set, reads the context flag byte again, clears bit `0x04` in that newly read value, and writes the resulting byte back.
4. Returns zero.

The flag-condition read occurs even when input bit `0x01` is clear. The update, when reached, preserves the other bits from the update read, not necessarily from the earlier condition read.

This path performs no second eligibility call, selector mapping, or table initialization.

### Second eligibility and selector requirement

On the in-range branch, the original selector is copied into mutable local 32-bit storage. F-3b037d560480 is then called again without arguments.

A zero result returns zero. If the gate succeeds but the selector copy is zero, the function also returns zero.

Neither return executes the exhausted-selection flag-clearing path.

The original selector is therefore not rejected before reference resolution or position calculation. With a nonzero reference, F-6fa4e01d8d29 may receive selector zero even though later table selection requires a nonzero selector before mapping.

### Second final-range check and mapping

After the second gate and selector requirement succeed, the function obtains the count again.

If the candidate is not below this new count, it returns zero without clearing context bit `0x04`.

Otherwise, F-3dfc1640191e receives:

- The context.
- The address of the mutable selector copy.
- The address of another local-storage region.

No return value from this dependency controls subsequent processing. The selector storage is used afterward to choose the table-pointer slot, so the dependency can change the selected slot.

There is no validation of the mapped selector. In particular, the earlier nonzero check does not reject a zero value produced by mapping.

### Reversal

With nonzero reverse control, the final position is:

**the count captured immediately before mapping, minus the candidate, minus one.**

With zero reverse control, the candidate is unchanged.

The second range check makes reversal valid relative to that captured count. There is no count reread or position revalidation after mapping.

If F-3dfc1640191e changes the descriptor, count, or associated storage, reversal still uses the earlier captured count. Whether such changes are permitted or safe is unknown.

### Table availability and result access

The function reads the table-pointer slot selected by the mapped selector.

- If nonzero, initialization is skipped.
- If zero, F-60950fd9e511 receives the context, and the selected slot is read again.
- If that reread is zero, the function returns zero.
- Otherwise, the selected slot is read again to form the entry address, and the 32-bit word at table + `0x10` + four times final position is returned.

The final access does not use a pointer guaranteed to be cached across the availability check. With an initially present table, the slot is read for availability and again for the result access. With successful initialization, it is read before initialization, after initialization, and again for the result access.

There is no eligibility, count, mapped-selector, or table-length check after initialization.

## Side effects

### Direct writes and observable order

The only explicit persistent-memory write is the conditional clearing of context bit `0x04` on the first final-range check’s exhausted-selection path.

The function also writes its local selector copy and exposes two local-storage addresses to F-3dfc1640191e.

Count observations are separate:

- A mode-one wrap-stage count, only when that path is enabled.
- A first final-range count.
- A second final-range count, after the second eligibility call and selector requirement.

Each nonzero-context count acquisition reads the descriptor address before reading the count. Later acquisitions do not reuse an earlier descriptor address or count. Reversal alone uses the count captured before mapping.

No synchronization, atomic update, or protection against concurrent mutation is visible. In particular, the context flag update is a read-and-write operation, not an established atomic bit clear.

### Dependency effects and ownership

Dependency implementations are unavailable. They must not be assumed to preserve context fields, descriptor addresses, counts, table contents, global state, or memory lifetime.

No allocation, release, ownership transfer, or reference-count operation is explicit in this function’s body. Dependencies may perform such operations.

The result is loaded from table storage; that fact establishes no ownership or release obligation. If the word encodes an address, its lifetime is **documented_only**.

The two addresses supplied to F-3dfc1640191e refer to local storage. Their size, permitted use, and retention rules are not fully established. Safe retention or asynchronous access must not be assumed.

### Errors

There is no distinct error status or visible invalid-address recovery. Zero can result from eligibility rejection, failed resolution, unsupported mode, range rejection, zero selector, unavailable table storage, or a zero-valued selected entry.

The visible body does not convert invalid memory accesses into zero returns.

## Dependencies

| Opaque ID | Established call-site contract |
|---|---|
| F-3b037d560480 | No arguments. Called first and conditionally again after the first successful range check. Zero rejects at either site. |
| F-6ef0016a253f | Receives context and nonzero reference. Zero rejects; a nonzero result is forwarded to position calculation. |
| F-6fa4e01d8d29 | Receives the resolution result and original selector. Supplies the unsigned 32-bit candidate. |
| F-8a6c49c858cd | Receives context and zero on a qualifying mode-one wrap. A nonzero result enables notification. |
| F-df1f4510064d | Receives context after that qualifying nonzero transition result. Its return does not control subsequent behavior. |
| F-3dfc1640191e | Receives context and two local-storage addresses. The selector storage is subsequently used for slot selection. |
| F-60950fd9e511 | Receives context when the selected table-pointer slot is zero. The slot is checked again afterward. |

“Resolution,” “transition,” “notification,” “mapping,” and “initialization” are inferred role labels, not independent specifications of these dependencies.

## Acceptance examples

These are conditional static examples, not executed tests. Except where an example specifies otherwise, assume valid memory, stable relevant state, successful eligibility calls, a nonzero selector unchanged by mapping, and a present table. **Mode-specific examples explicitly require context bit `0x08` to be clear.**

1. **Initial rejection.** If the first F-3b037d560480 call returns zero, the result is zero without a visible context-field read or later dependency call.

2. **Zero reference.** With reference zero, count five, and reversal disabled, position zero is selected. Mode handling and reference-related dependencies are skipped, regardless of the stored mode.

3. **Reversed initial selection.** With reference zero and count five at both final checks, nonzero reverse control selects position four.

4. **Mode-zero advancement.** With a nonzero reference, calculated position two, context bit `0x08` clear, mode zero, and count five, the candidate becomes three. Reversal disabled selects position three; reversal enabled selects position one.

5. **Mode-two flag distinction.** With a nonzero reference, calculated position two, context bit `0x08` clear, mode two, and count five, clearing input bit `0x04` selects position two; setting it selects position three, assuming reversal is disabled. If context bit `0x08` were instead set, both input-flag cases would advance to position three.

6. **Mode-one wrap and callbacks.** With a nonzero reference, calculated position four, context bit `0x08` clear, mode one, count five, and input bit `0x02` clear, advancement reaches five and wrapping resets the candidate to zero. Both `0x01` flags must be set to invoke F-8a6c49c858cd. F-df1f4510064d follows only on its nonzero result. Selection continues with newly read counts.

7. **Wrap suppression.** Under example 6’s prerequisites, including context bit `0x08` clear, set input bit `0x02`. The candidate remains five after advancement, no wrap-stage count or transition callback occurs, and the first final-range check rejects it. Both `0x01` flags being set cause context bit `0x04` to be cleared.

8. **Mode-one bypass.** Change example 6 by setting context bit `0x08`. The candidate advances from four to five without mode-one wrapping or callbacks. With count five, the first final-range check rejects it and conditionally clears bit `0x04`.

9. **Unsupported mode.** With a nonzero reference and context bit `0x08` clear, an unsupported mode returns zero before the final-range checks. With bit `0x08` set, that mode is not read and the calculated position advances once.

10. **Increment boundary.** A calculated position of `0xffffffff` becomes zero on an advancing path. With positive counts and all later requirements satisfied, position zero can be selected.

11. **Count changes at the second gate.** Candidate three passes a first count of four. If the second gate succeeds but the next count is three, the function returns zero without exhausted-selection flag clearing.

12. **Mapped selector zero.** A nonzero original selector passes the selector requirement. If mapping changes it to zero, the slot at context + `0x3ac` is used; mapping does not trigger another zero-selector rejection.

13. **Count changes during mapping.** Candidate one passes a second count of five. With reversal enabled, the final position is three even if mapping changes the descriptor count to two. Safe access then depends on the actual table storage; no revalidation occurs.

14. **Table availability and zero entries.** An absent selected table triggers one initialization call. A still-zero slot returns zero; a valid resulting table supplies the selected word. A selected word containing zero also returns zero.

## Uncertainties

The following remain **documented_only**:

- Dependency internals, additional preconditions, global effects, and failure mechanisms.
- Semantic meanings of reference values, selectors, modes, flags, and entries.
- Size, initialization requirements, and output meaning of the second local-storage region passed to mapping.
- Valid mapped-selector ranges and selector-derived address-overflow behavior.
- Endianness, alignment, allocation extents, and concurrency rules.
- Whether dependencies preserve storage needed by subsequent accesses.
- Ownership and lifetime of a returned address-like value.

These unknowns do not alter the established branch conditions, access widths, count-read timing, or absence of post-mapping validation.

## Evidence

This contract is based solely on the supplied static function representation and opaque dependency mapping. Established behavior follows from visible branches, call arguments, return checks, memory offsets, access widths, arithmetic, and access order.

Dependency role names and application-level meanings are identified as inferred. No dependency implementation, external ownership contract, or runtime trace was available. No runtime tests have been performed.
