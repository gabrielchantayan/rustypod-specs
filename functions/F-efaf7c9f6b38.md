<!-- Trusted provenance: F-efaf7c9f6b38; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-efaf7c9f6b38

## Purpose

F-efaf7c9f6b38 performs an ordered cache lookup and, on a miss, dispatches external operations before optionally inserting the supplied value into the cache.

The cache matches a Key and a signed Selector. Eligibility is an additional acceptance condition, not a third equality key. An override permits otherwise-ineligible matching entries to supply the result. The first qualifying entry wins.

On a miss, dispatch depends on the Selector’s sign, the presence of an Auxiliary object, and, when present, an auxiliary mode byte. Cache insertion is controlled by state read after the external operations.

**Established static behavior:** lookup order, dispatch arguments, conditional auxiliary processing, local memory-access order, and cursor/count maintenance.

**Inferred intent:** the state implements a bounded rotating cache. Application-level meanings of the dispatched operations are not established.

## Interface

The function accepts six arguments and returns a signed 32-bit integer.

| Argument | Representation and role |
|---|---|
| **State** | A 32-bit address-bearing value identifying mutable state, including a dependency context and inline cache storage. |
| **Auxiliary** | A 32-bit address-bearing value. Zero denotes absence; a nonzero value identifies an auxiliary object. |
| **Selector** | Signed 32-bit integer. Used in cache matching, dispatch selection, and, on one path, auxiliary-record indexing. |
| **Key** | Signed 32-bit integer used in cache matching and passed to the primary dependency. |
| **Supplied value** | Signed 32-bit integer returned on a normally completed miss, passed to the primary dependency, and optionally cached. |
| **Eligibility override** | Signed 32-bit zero/nonzero control. Any nonzero value permits a match whose eligibility test is nonzero. |

The supplied representation uses four-byte integers and four-byte address-bearing fields. No corresponding native layout for a wider-address environment is established.

### State layout

Offsets are in bytes from State.

| Offset | Width | Meaning |
|---|---:|---|
| `0x0C` | 4 bytes | Context captured before lookup and reused for dependency calls. |
| `0x54` | 4 bytes | Insertion suppression: zero permits local cache insertion; nonzero suppresses it. |
| `0x58` | 4 bytes | Signed searchable-entry count. |
| `0x5C` | 4 bytes | Signed write cursor when read for insertion. |
| `0x60` onward | 16 bytes per entry | Inline cache entries. |

Each entry has this layout:

| Entry-relative offset | Width | Meaning |
|---|---:|---|
| `0x00` | 4 bytes | Key. |
| `0x04` | 4 bytes | Selector. |
| `0x08` | 4-byte lookup read | Eligibility storage, narrowed to a character-sized value for comparison with zero. |
| `0x0C` | 4 bytes | Cached return value. |

Insertion clears only the byte at entry-relative offset `0x08`; it does not clear the remaining three bytes of the eligibility word. Under an eight-bit character representation, lookup tests the narrowed low eight bits of the loaded integer. Whether the byte cleared by insertion corresponds to those bits depends on target byte order and representation.

### Auxiliary layout

| Offset | Width | Meaning |
|---|---:|---|
| `0x08` | 4 bytes | Address of an auxiliary-record array. |
| `0x39` | 1 byte | Zero/nonzero mode indicator. |

Records have a stride of `0x14` bytes. The nonnegative-Selector path can inspect the byte at record-relative offset `0x12`.

### Preconditions and result

Every reached memory access requires valid, appropriately accessible storage. The function does not validate allocation sizes, alignment, record bounds, searchable count, or cursor.

Arithmetic used for indexing and address construction must remain representable under the supplied signed 32-bit operations. In particular, the record-index multiplication, cursor-based destination calculation, and signed cursor increment are not established wrapping operations.

External calls must return normally and leave any subsequently accessed storage valid for the local sequence to complete normally. Their preservation or ownership behavior is not otherwise assumed.

The result is either:

- The cached value from the first qualifying entry; or
- The original Supplied value after a normally completed miss.

There is no separate local error result.

## Behavior

### Context capture and cache lookup

The context at State offset `0x0C` is read before lookup, including on calls that subsequently return a cached value.

Lookup begins at entry zero. The searchable count at `0x58` is read at each loop condition and compared with the current index using signed comparison. It is not captured once for the entire search.

For each visited entry, reads are short-circuited in this order:

1. Read its Key.
2. If the Key matches, read its Selector.
3. If both match, read the eligibility word and narrow it for the zero test.
4. If eligibility is zero, or Eligibility override is nonzero, read and return the cached value.

The eligibility read and zero test precede consideration of the override. A nonzero override therefore does not eliminate that memory access.

An ineligible matching entry does not end lookup. A later eligible match can supply the result. Neither Supplied value nor Auxiliary participates in matching.

A hit performs no dependency calls and no local writes. It does not consult the insertion-suppression field or write cursor.

A searchable count of zero or less examines no entries. Positive counts are not clamped to ten or to any independently verified allocation size.

### Miss dispatch

All miss paths use the context captured before lookup. That context is not refreshed after a dependency call.

#### Negative Selector

For a negative Selector, the primary operation code is:

- `0x0B` if Auxiliary is present and its mode byte at `0x39` is nonzero.
- `0x27` if Auxiliary is absent or that mode byte is zero.

F-2ea344827b50 receives these five ordered arguments:

1. Captured context.
2. Selected operation code.
3. Key.
4. Supplied value.
5. Zero.

The negative Selector is not passed in the fourth argument position. This path does not locally read the auxiliary-record address and does not call the other two dependencies.

#### Nonnegative Selector, absent Auxiliary

F-2ea344827b50 receives:

1. Captured context.
2. Operation code `3`.
3. Key.
4. Selector.
5. Supplied value.

No auxiliary memory is accessed locally, and no other dependency is called before cache maintenance.

#### Nonnegative Selector, present Auxiliary

The mode byte at Auxiliary offset `0x39` is read first. Zero selects operation code `3`; nonzero selects `0x31`.

The ensuing sequence is:

1. F-2ea344827b50 receives captured context, selected operation code, Key, Selector, and Supplied value.
2. F-eec489a1f23b receives captured context, Auxiliary, and Selector.
3. The record-array address is read from Auxiliary offset `0x08`.
4. The byte at that address plus `Selector × 0x14 + 0x12` is read.
5. If that byte equals the character literal `e`, F-211638bb75da receives captured context and operation code `0x1C`.

The record-array address and selected byte are read after both preceding calls. Changes those calls make to the relevant storage can therefore affect the final conditional call, provided the storage remains valid.

The Selector’s nonnegativity does not establish a valid record index. Neither a record count nor an upper-bound check is present.

### Cache maintenance after a miss

Every normally returning miss-dispatch path proceeds to cache maintenance.

The suppression field at State offset `0x54` is read after the dependency sequence. If nonzero, there are no local cache writes. Dependencies can therefore affect the insertion decision by changing this field.

If suppression is zero, the signed cursor at `0x5C` selects the destination entry at State plus `0x60` plus sixteen times that cursor. The current cursor is used before any normalization.

The entry is written in this order:

1. Key at relative offset `0x00`.
2. Selector at relative offset `0x04`.
3. Supplied value at relative offset `0x0C`.
4. A zero byte at relative offset `0x08`.

There is no search for an existing entry to replace and no deduplication.

After these entry writes, cursor advancement has three distinct stages:

- Add one to the old cursor using **signed 32-bit arithmetic**.
- Convert the representable signed result to an unsigned 32-bit value.
- Compare that unsigned value with nine; replace it with zero if it is greater than nine.

The addition is not an unsigned wrapping increment. An old cursor of `INT_MAX` cannot be incremented within the signed 32-bit range and is outside the representable-arithmetic precondition. The function does not establish a wrap from that value to `0x80000000`.

For a representable negative increment result, conversion to unsigned precedes the comparison. Such a converted value exceeds nine and is normalized to zero. An increment result of zero also produces zero. These arithmetic facts do not make a negative cursor safe: destination addressing and entry writes occur earlier.

After normalization, the next cursor is between zero and nine. The function reads the current signed searchable count and compares it with that next cursor interpreted as signed. If the count is smaller, it writes the next cursor into the count field. It then writes the next cursor into the cursor field.

A normally completed miss returns the original Supplied value. Dependency return values, if any, are not used.

### Cursor and searchable-count boundaries

For old cursors zero through nine, advancement produces one through nine, followed by zero.

The searchable count is raised to the normalized next cursor only when the current signed count is smaller. It is not updated directly to the number of physical entries written.

Starting with count zero and cursor zero, assuming enabled insertion and no interfering changes:

- The first insertion writes entry zero and sets count and cursor to one.
- The ninth insertion writes entry eight and sets count and cursor to nine.
- The tenth insertion writes entry nine, sets cursor to zero, and leaves count at nine.

With count nine, lookup examines entries zero through eight. Thus, the ordinary sequence does not make entry nine searchable. Conversely, an independently established count of ten would cause lookup to examine entry nine; the lookup itself imposes no ten-entry cap.

## Side effects

### Local memory effects

Local reads comprise context capture, repeated count checks, conditionally selected entry fields, path-dependent auxiliary fields, and post-call insertion controls.

Local writes occur only after a miss with post-call suppression equal to zero. Their order is entry Key, entry Selector, entry Supplied value, eligibility byte, optional count update, and cursor update.

The signed cursor increment occurs after the entry writes. Arithmetic failure or invalid storage is not represented by a recoverable local status, and no rollback is shown.

No local allocation, deallocation, or explicit ownership transfer is established.

### Dependency effects

Dependency effects on memory, globals, ownership, resources, and control flow are unknown. In particular, they are not assumed to preserve State, Auxiliary, cache controls, record addresses, or globals. Nor are they assumed to retain or release any resource.

Later accesses use the original State and Auxiliary address values and the originally captured context, but can observe changed contents at those addresses.

### Synchronization

No synchronization or atomic multi-field publication is established. The ordered entry writes do not constitute a demonstrated thread-safe update protocol. Repeated count reads likewise do not establish concurrency safety.

## Dependencies

| Opaque function ID | Established use |
|---|---|
| **F-2ea344827b50** | Primary dispatch, called once on each miss that reaches dispatch. Takes five ordered arguments whose operation code and final two positions depend on the selected path. |
| **F-eec489a1f23b** | Called after primary dispatch only for a miss with nonnegative Selector and present Auxiliary. Takes captured context, Auxiliary, and Selector. |
| **F-211638bb75da** | Called after the preceding two operations only when the selected auxiliary-record byte equals `e`. Takes captured context and `0x1C`. |

A cache hit calls none of these dependencies. Their broader semantics are **documented_only** unknowns.

## Acceptance examples

These are conditional static acceptance criteria, not observed runtime results. Unless stated otherwise, all reached storage and arithmetic must satisfy the preconditions, and dependencies must return normally.

### Cached result takes precedence

Given a searchable matching entry with zero eligibility and cached value 42, a call with Supplied value 99 returns 42. No dependency is called, and no local cache write occurs.

### Eligibility override changes which duplicate wins

Given two searchable matching entries, the first with nonzero eligibility and the second with zero eligibility:

- Override zero selects the second entry’s cached value.
- Override nonzero selects the first entry’s cached value.

The first entry’s eligibility word is read in both cases.

### Negative Selector changes argument placement

Given a miss, Selector −1, absent Auxiliary, and Supplied value 17, F-2ea344827b50 receives captured context, `0x27`, Key, 17, and zero. A normally completed call returns 17.

With present Auxiliary, a nonzero mode byte changes the operation code to `0x0B`; it does not cause record indexing.

### Nonnegative Selector preserves its dispatch position

Given a miss, nonnegative Selector, and absent Auxiliary, F-2ea344827b50 receives captured context, `3`, Key, Selector, and Supplied value. No auxiliary-processing dependency is invoked.

### Auxiliary processing uses post-call records

Given a miss with nonnegative Selector and present Auxiliary, primary dispatch precedes F-eec489a1f23b. If those calls leave a valid selected record whose inspected byte equals `e`, F-211638bb75da follows with `0x1C`. Otherwise that final call is absent.

The primary operation code is determined by the mode byte read before either call.

### Post-call suppression controls insertion

If suppression is initially zero but a dependency changes it to nonzero, insertion is skipped. Conversely, if dependencies leave it zero after changing an initially nonzero value, insertion proceeds.

These outcomes require the state to remain valid.

### The tenth write does not increase an ordinary count of nine

After nine enabled miss insertions from count zero and cursor zero, count and cursor are nine. The next miss writes entry nine, wraps the cursor to zero, and leaves count nine. A subsequent lookup examines only entries zero through eight.

### Signed-increment overflow has no promised wrap result

An old cursor equal to `INT_MAX` is outside the signed-increment precondition. No acceptance criterion may require cursor wrap, successful insertion, or a normal return for that state. Destination arithmetic and memory validity are independent prerequisites that may already fail before the increment is reached.

## Uncertainties

- **Dependency semantics — documented_only:** Operation meanings, ownership effects, global mutations, and failure behavior are unavailable.
- **Target representation — documented_only:** Native wider-address layouts are not established. The relationship between the cleared eligibility byte and the narrowed lookup value requires byte-order and character-representation information.
- **State invariants — documented_only:** No lifecycle contract establishes permissible initial counts, cursors, or allocation sizes.
- **Record bounds — documented_only:** No record-array length or Selector upper bound is supplied.
- **Invalid arithmetic or storage — documented_only:** No portable behavior is promised outside representable signed arithmetic and valid-access preconditions. In particular, signed cursor overflow is not converted into a defined unsigned increment by its later assignment.
- **Character encoding — documented_only:** The comparison is to `e`; an independent numeric encoding is not supplied.
- **Cache design intent — inferred:** Ten rotating write positions are suggested by normalization, but ordinary count maintenance exposes only nine entries. Whether this is intentional is unknown.
- **Runtime behavior — unverified:** No runtime tests have been performed.

## Evidence

The supplied static representation establishes:

- Context capture before lookup and immediate return on the first qualifying match.
- Signed count-controlled lookup with ordered, short-circuit field reads.
- Miss dispatch selected by Selector sign, Auxiliary presence, and a mode byte.
- Ordered auxiliary processing with record reads after two dependency calls.
- Post-call suppression and cursor reads.
- Three full-width entry writes followed by a one-byte eligibility clear.
- A **signed 32-bit cursor increment, then unsigned conversion, unsigned comparison against nine, and normalization**.
- Signed count comparison after normalization, followed by conditional count storage and unconditional cursor storage when insertion is enabled.
- Return of the original Supplied value on every normally completed miss.

No dependency definitions, caller invariants, allocation sizes, independent target-format declaration, execution traces, or runtime verification were supplied. Those limits remain unresolved rather than being replaced with assumptions.
