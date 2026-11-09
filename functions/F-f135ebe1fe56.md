<!-- Trusted provenance: F-f135ebe1fe56; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-f135ebe1fe56

## Purpose

F-f135ebe1fe56 conditionally processes an existing object's pending state, issues a sequence of dependency calls against one or two resolved targets, and clears selected fields on completion.

The established behavior has three distinct outcomes:

- **Entry rejection:** return immediately without calling dependencies or explicitly modifying the object.
- **Final dispatch:** invoke F-b56a17d0278a, set the object's status to 101, clear one byte, then perform common cleanup.
- **Fallback completion:** change a zero status to 1, preserve a nonzero status, then perform common cleanup.

The meanings of the dispatched operation codes, target objects, status values, and pending records are not established. Descriptions such as “pending state” identify their structural role, not a proven application-level meaning.

## Interface

The function has four arguments and no returned value:

1. **Object address:** address of mutable storage containing 32-bit fields, byte fields, addresses, counts, masks, and record references.
2. **Opaque 32-bit value:** not used by the supplied body.
3. **Opaque 32-bit value:** initially supplied by the caller; may be replaced during record processing before being passed to the final dispatch.
4. **Opaque 32-bit value:** initially supplied by the caller; may similarly be replaced.

Offsets below are byte offsets. All listed word fields occupy 32 bits. Addresses and indexed address-table entries are represented as 32-bit values in the supplied representation.

### Main object layout

| Offset | Access and role |
|---|---|
| `0x000` | Address of a referenced context. Captured once at entry. |
| `0x004` | Status word. Compared with zero; potentially assigned 1 or 101. |
| `0x010` | One byte cleared after successful final dispatch. |
| `0x013` | Entry-inhibition byte. Read only when the context's `0x01E` byte is zero. |
| `0x040` | Entry-inhibition word, also rechecked before final dispatch. |
| `0x044` | Coordinate-like word passed to final dispatch after adding 3; later cleared. |
| `0x048` | Another coordinate-like word, likewise adjusted and cleared. |
| `0x04C` | Word cleared during common cleanup. |
| `0x100` | Bit mask supplying Boolean values for selected indices. |
| `0x104` | Bit mask selecting indices for three per-index calls; later cleared. |
| `0x108` | Signed processing value. Positive values enable the extended processing block; later cleared. |
| `0x10C + 4 × index` | Per-index 32-bit values used for operation `0x60`. |
| `0x13C` | Signed count for a second record sequence. |
| `0x140` | Base address of that sequence's 16-byte records. |
| `0x14C` | Opaque word passed to final dispatch; later cleared. |
| `0x15C` | Word whose conversion to a character-sized value is passed to final dispatch. |
| `0x19C` | Signed count for a first record sequence; explicitly set to zero after traversal. |
| `0x1A0` | Base address of a table of 32-bit record addresses. |

The indexed region beginning at `0x10C` has no separately established capacity. Sufficiently large indices overlap other listed fields. The source does not establish a safe maximum count or an independent array boundary.

The bytes at `0x010` and `0x013` belong to the same four-byte span, but clearing `0x010` does not clear `0x013`. No byte order is assumed.

### Referenced context and record layouts

The context captured from object offset `0x000` supplies:

- A signed 32-bit iteration bound at offset `0x004`.
- An entry-inhibition byte at offset `0x01E`, re-read before final dispatch when applicable.
- A byte at offset `0x080` controlling a later query.

Each first-sequence table entry addresses a record with a 32-bit payload at offset `0x040`.

Each second-sequence record occupies 16 bytes:

| Record offset | Use |
|---|---|
| `0x00` | First 32-bit payload |
| `0x04` | Second 32-bit payload |
| `0x08` | One-byte payload |
| `0x09`–`0x0B` | Not explicitly read |
| `0x0C` | Third 32-bit payload |

### Preconditions

All storage reached along the executed path must be valid for the corresponding reads and writes. The function performs no explicit null check on the main object or captured context, no record-table bounds checks, and no capacity checks.

Dependencies must return normally for subsequent behavior to occur. Their effects on object storage, referenced storage, or global state are not constrained by this body. No ownership transfer, retention, release, or lifetime guarantee is established.

## Behavior

### 1. Entry gate

The function first captures the context address from the main object. Later context accesses continue to use this captured address, even if a dependency changes the object's context-address field.

It then applies a short-circuit gate in this order:

1. Read the context byte at `0x01E`.
2. Only if that byte is zero, read the object byte at `0x013`.
3. Only if both bytes are zero, read the object word at `0x040`.

Processing continues only when all three values are zero. Otherwise the function returns immediately.

This immediate-return path performs no dependency calls and no explicit object writes. It does not perform common cleanup.

### 2. Initial target resolution

The function calls F-9b3ea8470521 with the main object address.

A zero result bypasses all target-dependent processing and proceeds to fallback status handling and common cleanup.

A nonzero result becomes the **primary target**. The function first calls F-7f44e0e8f855 with that target and `0x26`.

The object word at `0x108` is tested for signed positivity after this call. Thus, a change made by the dependency can affect whether extended processing runs.

### 3. Extended processing

This block runs only if the then-current signed word at `0x108` is greater than zero.

First, F-2f5fbad4fdef receives:

- The primary target.
- The current value at `0x108`, minus 1.

The positive condition makes the subtraction nonnegative for an unchanged ordinary signed 32-bit value.

#### Selected-index processing

The function traverses indices starting at zero, comparing each index against the signed word at context offset `0x004`.

The bound is read again on each loop test. It is not a fixed entry snapshot.

A 32-bit unsigned selection bit starts at 1. At each index, the function tests the current object mask at `0x104`. If the current bit is set, it issues, in order:

1. F-4972fa49e746 with the primary target and index.
2. F-887ba2404a49 with the primary target, operation `99` (`0x63`), the index, and a Boolean indicating whether the corresponding bit is set in the current mask at `0x100`.
3. F-887ba2404a49 with the primary target, operation `0x60`, the index, and the current word at `0x10C + 4 × index`.

These are separate calls. Values read for later calls can reflect effects of earlier calls. The body does not snapshot the masks or indexed values for the entire traversal.

After each iteration, the selection bit is shifted left once as a 32-bit unsigned value. Indices 0 through 31 have representable selection bits. From index 32 onward, the selection bit is zero, so no index is selected by this mask mechanism, although loop tests continue if the context bound remains larger.

A zero or negative bound produces no per-index calls.

#### First record sequence

Next, indices start again at zero and are compared against the current signed count at object offset `0x19C`.

For each visited index:

- Read the table base at `0x1A0`.
- Read the record address from the table's four-byte entry for that index.
- Read the 32-bit payload at record offset `0x040`.
- Replace the effective third argument with that payload.
- Replace the effective fourth argument with `0xFFFFFFF6`.
- Call F-0b0cd8d831b7 with the primary target and the ordered argument tuple: `0x76`, `0`, `0`, `0`, payload, `0xFFFFFFF6`.

The count and table base are not guaranteed to remain constant across calls.

After traversal, the function writes zero to object offset `0x19C`, including when the initial count was zero or negative. This does not establish that any record or table memory is released.

#### Second target resolution and record sequence

The function calls F-9b3ea8470521 again with the main object address. This result is independent of the primary target; equality between the two is not required.

If this second result is zero, second-sequence traversal is skipped.

Otherwise, indices start at zero and are compared against the current signed count at `0x13C`. For each visited 16-byte record, the function:

- Reads the record base from `0x140`.
- Replaces the effective third argument with the record's word at `0x0C`.
- Replaces the effective fourth argument with `0xFFFFFFFE`.
- Calls F-0b0cd8d831b7 with the second target and the ordered argument tuple: `0x5C`, record word at `0x00`, record word at `0x04`, record byte at `0x08`, record word at `0x0C`, `0xFFFFFFFE`.

The byte payload is read as an unsigned one-byte quantity in the supplied representation.

The count and base can reflect dependency-induced changes between iterations. This function does not explicitly clear the second-sequence count or base.

Whether or not the second target resolved, extended processing ends by calling F-887ba2404a49 with the primary target, operation `0x5B`, index-like argument zero, and the current word at `0x108`.

### 4. Conditional query and action

After extended processing—or immediately after skipping it—the function reads the captured context's byte at `0x080`.

If that byte is nonzero, no query occurs.

If it is zero, the function calls F-b05552b91301 with the primary target and zero. A zero result ends this conditional action. A nonzero result is treated as an address of readable byte storage.

If the first byte at that address equals `0x14`, the function calls F-2b905b6e3991 with the primary target and zero. Other byte values produce no such call.

The body does not release or retain the returned storage explicitly.

### 5. Final eligibility check and dispatch

Before final dispatch, the function rechecks:

1. The current object word at `0x040`.
2. Only if that word is zero, the captured context's current byte at `0x01E`.

Both must be zero. The entry-inhibition byte at object offset `0x013` is **not** rechecked.

If eligible, F-b56a17d0278a receives, in order:

- The primary target.
- The current object word at `0x14C`.
- The current word at `0x048`, plus 3.
- The current word at `0x044`, plus 3.
- The word at `0x15C` converted to a character-sized argument.
- The effective third argument.
- The effective fourth argument.

The two coordinate-like additions are distinct 32-bit signed additions in the supplied representation. Behavior at signed overflow is not established.

The effective last two arguments depend on earlier traversal:

| Last applicable event | Effective third argument | Effective fourth argument |
|---|---|---|
| No record iteration occurred | Original third argument | Original fourth argument |
| First sequence processed records, with no later second-sequence iteration | Last first-sequence payload | `0xFFFFFFF6` |
| Second sequence processed records | Last second-sequence word at `0x0C` | `0xFFFFFFFE` |

After F-b56a17d0278a returns, the function writes status 101 to `0x004`, clears the byte at `0x010`, and proceeds directly to common cleanup.

### 6. Fallback and common cleanup

Fallback applies when initial target resolution returns zero or when the final eligibility check fails.

On fallback:

- If the current status at `0x004` is zero, write 1.
- Otherwise preserve that status.
- Do not explicitly clear the byte at `0x010`.

All paths that passed the entry gate then perform these word writes, in order:

1. Zero `0x044`.
2. Zero `0x048`.
3. Zero `0x04C`.
4. Zero `0x14C`.
5. Zero `0x104`.
6. Zero `0x108`.

These cleanup writes occur even when initial target resolution failed. The mask at `0x100`, the word at `0x040`, and the second-sequence count and base are not explicitly cleared.

## Side effects

Established direct effects consist of dependency calls, status changes, the conditional one-byte clear, the first-sequence count reset within extended processing, and common cleanup.

The function contains no explicit allocation, deallocation, reference-count update, record destruction, or error return. Clearing a count or address-like payload must not be interpreted as releasing its referents.

Dependency effects remain unknown. In particular:

- Dependencies may change fields read later.
- The second resolution may return a different target.
- Dependency changes can enable or inhibit final dispatch.
- Explicit post-call writes can overwrite dependency changes to the same fields.
- A dependency failure that does not return normally can prevent later cleanup; no recovery mechanism is shown.

Ordering is established between separate statements, calls, and explicit writes. The source does not establish the relative evaluation order of multiple memory-derived arguments within one call.

## Dependencies

| Opaque ID | Established use |
|---|---|
| F-9b3ea8470521 | Resolves a target from the main object; zero means unavailable for the corresponding branch. Called once, and again only within extended processing. |
| F-7f44e0e8f855 | Receives the primary target and `0x26`. |
| F-2f5fbad4fdef | Receives the primary target and the positive processing value minus 1. |
| F-4972fa49e746 | Receives the primary target and a selected index. |
| F-887ba2404a49 | Receives operation `99`, `0x60`, or `0x5B` with the associated index and value. |
| F-0b0cd8d831b7 | Receives first- or second-sequence record payloads with operation `0x76` or `0x5C`. |
| F-b05552b91301 | Queries the primary target with zero and returns either zero or a byte-readable address. |
| F-2b905b6e3991 | Invoked when the query result's first byte is `0x14`. |
| F-b56a17d0278a | Receives the final payload, adjusted coordinate-like values, character-sized value, and effective trailing arguments. |

No broader dependency contracts are supplied.

## Acceptance examples

These are conditional static expectations, not executed tests. Unless stated otherwise, dependencies return normally and do not modify relevant storage.

1. **Rejected entry:** The captured context byte at `0x01E` is nonzero. No dependency is called, status remains unchanged, and none of the cleanup fields is cleared. The object byte at `0x013` and word at `0x040` need not be read.

2. **Unavailable primary target:** All entry gates are zero; initial resolution returns zero; status initially equals zero. No other dependency is called. Status becomes 1, and all six common-cleanup words become zero. The first-sequence count is not explicitly reset on this path.

3. **Dispatch without extended processing:** Entry succeeds, resolution returns a nonzero target, and `0x108` is zero. If the context's `0x080` byte is nonzero and final eligibility remains satisfied, no query or record-processing call occurs. Final dispatch receives the caller's original third and fourth arguments. Status becomes 101 and byte `0x010` becomes zero.

4. **Selected indices:** Extended processing is enabled, the context bound is 3, and mask `0x104` is binary `101`. Only indices 0 and 2 produce per-index calls, each in the three-call order specified above. Index 1 produces none. If the bound instead extends past 31, indices 32 and above still cannot be selected by the exhausted 32-bit selection bit.

5. **First-sequence trailing arguments:** One first-sequence record supplies payload `0x12345678`; the second resolution returns zero. If final dispatch occurs, its last two arguments are `0x12345678` and `0xFFFFFFF6`. The first-sequence count has been explicitly set to zero.

6. **Second-sequence precedence:** Both sequences process records, and the final second-sequence record has `0x89ABCDEF` at offset `0x0C`. Final dispatch receives `0x89ABCDEF` and `0xFFFFFFFE`, regardless of the last first-sequence payload.

7. **Late inhibition:** A dependency changes object word `0x040` to nonzero. Final dispatch is skipped; the context byte at `0x01E` is not needed for that final decision. A nonzero current status is preserved, but common cleanup still occurs.

8. **Entry byte not rechecked:** A dependency changes object byte `0x013` to nonzero while final-check fields remain zero. That change alone does not block final dispatch.

## Uncertainties

- **Documented_only:** Application-level meanings of fields, operation codes, targets, records, and status values.
- **Documented_only:** Dependency ownership rules, global effects, reentrancy, and failure behavior.
- **Documented_only:** Signedness and promotion details of the character-sized final-dispatch argument for values with the high bit set.
- **Documented_only:** Behavior when signed additions, index increments, or address-offset calculations overflow.
- **Documented_only:** Valid capacities of indexed storage and record tables, including whether layout overlap is intentional.
- **Documented_only:** Concurrency and volatile-memory semantics. The static read sequence does not establish synchronization guarantees.
- **Inferred intent:** The function appears to consume or submit accumulated state and mark completion. The evidence establishes the calls and writes, not that interpretation.

## Evidence

This contract is derived solely from the supplied function body and opaque dependency mapping. Static evidence establishes branch conditions, access widths, offsets, explicit write order, call order, loop structure, and argument substitution.

Dependency implementations, surrounding type definitions, callers, and runtime traces were not supplied. No runtime tests have been performed. The acceptance examples describe consequences of the stated conditions rather than observed executions.
