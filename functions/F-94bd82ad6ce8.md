<!-- Trusted provenance: F-94bd82ad6ce8; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-94bd82ad6ce8

## Purpose

F-94bd82ad6ce8 updates a cached position associated with a sequence of signed 32-bit values. It accepts a context address and a signed 32-bit target.

The established behavior has three paths:

1. If the sequence’s final value is less than or equal to the target, store the final position and return.
2. Otherwise, if an existing nonzero cached position brackets the target with a strictly smaller predecessor and a greater-than-or-equal current value, return without writing the cache.
3. Otherwise, clear the cache, search forward from position zero, store the resulting position, and transfer control to an unresolved indirect dependency.

For a nonempty, nondecreasing sequence whose storage remains stable, the selection behavior is consistent with finding the earliest value greater than or equal to the target, capped at the final position. This is **inferred intent**, not an unconditional contract: the final-value shortcut takes precedence and can select a later duplicate, and the function neither validates ordering nor repairs invalid metadata.

The indirect dependency is reached only on the third path. Its effects are unknown and must not be treated as equivalent to a normal return with no additional work.

## Interface

### Inputs

The supplied representation exposes two inputs:

- **Context address:** a 32-bit integer-valued address used to access context storage.
- **Target:** a signed 32-bit integer used in all value comparisons.

The source-level representation does not establish a portable calling convention, native pointer ABI, argument-register assignment, or alignment policy. The address widths stated here describe the supplied 32-bit address-bearing representation.

### Context and descriptor layout

All offsets are byte offsets. Positions are zero-based.

| Storage | Offset | Meaning |
|---|---:|---|
| Context | `0x40` | Reference to a sequence descriptor |
| Context | `0xA4` | Signed 32-bit cached position |
| Descriptor | `0x00` | Signed 32-bit element count |
| Descriptor | `0x04` | 32-bit integer value interpreted as the sequence base address |
| Sequence | `4 × position` | Signed 32-bit element |

Sequence elements have a four-byte stride. The descriptor’s count and base-address entries occupy consecutive four-byte slots in the supplied representation.

The descriptor reference is fetched from the context once during the visible body. The sequence base is initially fetched from the descriptor and retained for the final-element check and cached-position checks. The fallback search instead rereads the descriptor’s sequence-base entry for each element comparison that it reaches.

### Output

There is no explicit return value.

The visible result is a possible update to the context’s cached-position field, followed either by direct return or by an unresolved indirect transfer. If that dependency returns normally, the visible body performs no further explicit work before returning. Whether the dependency actually returns is unknown.

### Preconditions for a defined, ordinary execution

The caller must provide storage valid for every access reached on the chosen path:

- A readable context descriptor-reference field.
- A readable descriptor count and sequence-base entry.
- A readable final sequence element.
- A readable cached-position field if the final-value shortcut is not taken.
- Readable sequence elements addressed by any cached-position check or search iteration.
- A writable cached-position field whenever a store is reached.
- A valid target for the unresolved indirect transfer if the fallback path reaches it.

A nonempty sequence, valid cached position, stable metadata, and representable address arithmetic form the ordinary supported case. None of these properties is checked.

A stale cached position need not be valid on the final-value shortcut: that path overwrites it without reading it.

## Behavior

### 1. Initial descriptor and final-element access

The function first obtains the sequence descriptor from the context. It then obtains the descriptor’s sequence-base address and reads the element addressed as the final element: one four-byte slot before the address obtained by advancing the base by the element count.

This read occurs before any empty-sequence check; no such check exists.

The final element is compared with the target using signed 32-bit ordering.

If the final element is less than or equal to the target:

- The descriptor’s count is read again.
- That newly read count minus one is stored in the context’s cached-position field.
- The function returns directly.
- The old cached position is not read.
- The indirect dependency is not invoked.

The count used for the stored position is therefore not necessarily the same observed count used to address the final element. Equality follows only when the descriptor remains stable.

### 2. Cached-position reuse

If the initially read final element is greater than the target, the function reads the current cached position.

For a nonzero cached position, it reads the preceding sequence element using the initially retained sequence base. The supplied comparison accounts for signed subtraction overflow when determining whether that predecessor is strictly less than the target. It must therefore be understood as a signed comparison, not as an unchecked subtraction followed solely by a sign test.

Only if the predecessor is strictly less than the target does the function read the element at the cached position and test whether the target is less than or equal to that element.

If both comparisons succeed, the function returns directly:

- The cached position is preserved.
- No explicit memory write occurs in this function.
- The fallback search is skipped.
- The indirect dependency is not invoked.

This path recognizes the interval:

**predecessor < target ≤ current element**

Equality with the predecessor does not qualify.

A cached position of zero never qualifies for this early return. In that case, the function does not read a predecessor and does not read the current element as part of the cache-reuse check. It proceeds to the fallback path.

No range validation protects a nonzero cached position. Negative positions and positions beyond the sequence can cause out-of-range reads before fallback is considered. The current-element read is nevertheless conditional: it is skipped if the predecessor comparison fails.

### 3. Fallback search

When cache reuse does not succeed, the function first writes zero to the cached-position field.

It then searches from position zero. Each iteration evaluates its conditions in this order:

1. Read the descriptor’s current count and determine whether the candidate position is strictly less than that count minus one.
2. Only if that bound condition succeeds, reread the descriptor’s sequence-base entry and read the element at the candidate position.
3. Advance to the next position only if the element is strictly less than the target.

The search stops when either:

- The candidate is not less than the currently observed count minus one; or
- The compared element is greater than or equal to the target.

The resulting candidate is then stored in the cached-position field.

The initial zero store and the final result store are separate operations in the supplied behavior. When the result is zero, both stores write zero. This contract does not claim a particular number of physical bus transactions or volatile accesses.

The fallback does not necessarily read the final element again. For a stable positive count, reaching the final position makes the bound condition fail before the element-comparison operand is evaluated. The earlier final-element read remains part of the execution.

### 4. Indirect transfer

After storing the fallback result, the function transfers control through an unresolved indirect target, identified here as **F-U01**.

F-U01 is invoked even if the resulting position equals the position present on entry. Invocation depends on reaching the fallback path, not on whether the cached value changes.

There is no explicit cache store after this transfer. Consequently, the stored search result is established immediately before invoking F-U01, but it is not guaranteed to remain the final observable cache value afterward.

### Stable, ordered-sequence interpretation

For a stable, positive count and a nondecreasing sequence:

- A target at or above the final value selects the final position immediately.
- Otherwise, an existing nonzero cache is retained if its predecessor is strictly below the target and its current element is at least the target.
- Otherwise, the forward search selects the first element greater than or equal to the target.

Duplicate values matter. If the target equals the final value, the final-value shortcut selects the last position even when an earlier equal value exists. If the target is below the final value, fallback selects the first qualifying position under the stable, ordered-sequence assumptions.

## Side effects

### Explicit reads

The visible body reads:

- The context’s descriptor reference.
- The descriptor’s sequence-base entry and count.
- The initially addressed final element.
- The cached position, except on the final-value shortcut.
- A predecessor and possibly the current cached element, when the cache is nonzero.
- Repeated descriptor counts and conditionally reread sequence-base entries during fallback.
- Sequence elements reached by the fallback comparisons.

Short-circuiting affects which accesses occur. In particular:

- A failed predecessor comparison suppresses the cached current-element read.
- A failed fallback bound condition suppresses the sequence-base reread and candidate-element read for that iteration.

### Explicit writes

Only the context’s cached-position field is explicitly written:

| Path | Explicit stores before any dependency effects |
|---|---|
| Final-value shortcut | One store of the newly read count minus one |
| Successful cache reuse | None |
| Fallback | Zero, then the computed candidate |

The visible body does not explicitly modify sequence elements, descriptor entries, or the context’s descriptor reference.

### Ownership and lifetime

No explicit allocation, deallocation, ownership transfer, or reference-count operation appears in the visible body.

This establishes no ownership behavior for F-U01. It may modify state or release storage; those possibilities cannot be excluded from the available evidence. The caller must keep accessed storage valid through the last access that requires it.

### Errors and exceptional conditions

There is no explicit status result, validation failure, recovery path, or error report.

Invalid storage, invalid indices, invalid arithmetic, or an invalid indirect target have no defined recovery behavior in the supplied evidence. It would be incorrect to specify them as silently ignored or converted into a particular error.

Concurrent mutation is not synchronized here. Descriptor changes can affect later reads without changing earlier decisions. The retained initial base and the fallback’s reread base may also refer to different storage. No coherent snapshot is guaranteed.

## Dependencies

### F-U01 — unresolved indirect dependency

F-U01 is a document-local opaque identifier for the unresolved transfer target, not an established identity for a known implementation.

Established properties:

- Reached only after fallback finishes.
- Reached after the final explicit cached-position store.
- Not reached by either direct-return path.
- No explicit arguments are shown in the supplied representation.

Unknown properties include its actual calling convention, inputs carried by machine state, return behavior, memory effects, ownership effects, global-state changes, and failure behavior.

No claim is made that F-U01 preserves the context, descriptor, sequence, globals, or any allocated storage.

No other explicit callable dependency is established by the supplied body.

## Acceptance examples

These examples are conditional static expectations, not executed tests. Unless stated otherwise, metadata and sequence storage remain stable, arithmetic is representable, all required addresses are valid, and the sequence is nondecreasing. For fallback examples, the stated cache result is the value immediately before F-U01.

1. **Target above the final value**

   Sequence: `[10, 20, 30]`; target: `35`; initial cache: arbitrary.

   Expected: read final value `30`, store position `2`, and return directly. Do not read the old cache or invoke F-U01.

2. **Equality with a duplicated final value**

   Sequence: `[10, 20, 20]`; target: `20`; initial cache: arbitrary.

   Expected: store position `2` through the final-value shortcut, not position `1`.

3. **Successful cached interval**

   Sequence: `[10, 20, 30]`; target: `15`; initial cache: `1`.

   Expected: after the final-value check fails, establish `10 < 15 ≤ 20`; return without writing the cache or invoking F-U01.

4. **Predecessor equality rejects reuse**

   Sequence: `[10, 20, 30]`; target: `10`; initial cache: `1`.

   Expected: predecessor equality fails the strict lower comparison. Do not read position `1` for the second cache predicate. Write zero, select position `0` during fallback, store zero again, and invoke F-U01.

5. **Zero cache forces fallback**

   Sequence: `[10, 20, 30]`; target: `5`; initial cache: `0`.

   Expected: no cached predecessor or current-element check. Perform both zero stores and invoke F-U01.

6. **Fallback advances across smaller elements**

   Sequence: `[10, 20, 30, 40]`; target: `25`; initial cache: `1`.

   Expected: cache reuse fails because `25` exceeds `20`. Fallback passes positions `0` and `1`, stops at position `2`, stores `2`, and invokes F-U01.

7. **Single-element sequence**

   Sequence: `[7]`.

   For target `7`, select position `0` and return through the final-value shortcut. For target `6` with initial cache `0`, fallback stores zero without a loop element read and invokes F-U01.

8. **Signed comparison across the subtraction boundary**

   Sequence: `[−2147483648, 2147483647]`; target: `1`; initial cache: `1`.

   Expected: retain position `1` through cache reuse. The mathematical predecessor difference is outside signed 32-bit range, but the supplied overflow-aware comparison still establishes that the predecessor is below the target.

## Uncertainties

- **documented_only — Indirect target:** F-U01 cannot be resolved from the supplied evidence.
- **documented_only — Native ABI:** Physical pointer width, calling convention, and precise indirect-transfer mechanics are not established independently of the supplied representation.
- **documented_only — Invalid arithmetic:** No portable behavior is specified for overflowing address calculations, count-minus-one arithmetic, or search-position increments.
- **documented_only — Empty or negative counts:** The function does not reject them and attempts its initial final-element read before any protective branch. No valid empty-sequence result is established.
- **documented_only — Unordered data:** The branch mechanics remain as described, but no meaningful ordered-search result is guaranteed.
- **documented_only — Aliasing and mutation:** Context, descriptor, and sequence overlap are not prohibited by visible checks. The stable-layout examples do not cover overlapping storage or mutation between accesses.
- **Inferred intent:** The cache likely represents a target interval or selected sequence boundary. The application-level meaning is not established.

## Evidence

This contract is based solely on static inspection of the supplied function representation.

The evidence establishes the two context offsets, four-byte descriptor slots and element stride, signed value comparisons, overflow-aware predecessor comparison, short-circuit access conditions, three control-flow paths, explicit cache stores, repeated metadata reads, and unresolved indirect transfer.

It does not establish whole-program invariants, valid caller inputs, sequence ordering, ownership rules, dependency effects, or native runtime behavior.

No runtime tests have been performed.
