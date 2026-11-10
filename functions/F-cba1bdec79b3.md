<!-- Trusted provenance: F-cba1bdec79b3; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-cba1bdec79b3

## Purpose

F-cba1bdec79b3 dispatches six operations on a record whose address is the containing-object address plus `0x18`.

Three operations adjust a time-related quantity through conversion dependencies. Three operations snapshot the record, edit selected packed fields in temporary storage, consult a comparison dependency, and conditionally invoke a transfer dependency and a notification dependency.

**Established static behavior:** the 64-bit result contains a completion indicator in its low 32 bits and a path-dependent payload in its high 32 bits. Unsupported selectors and calendar paths whose comparison result is zero return completion zero. Time paths and calendar paths that reach notification return completion one if all calls return normally.

**Inferred intent:** the record represents calendar and time information. “Hour,” “minute,” “day,” “month,” and “year” below identify apparent field roles, not independently established calendar semantics.

The contract distinguishes packed-word bit positions from memory byte offsets. The supplied body does not establish the byte order used to store 32-bit words.

## Interface

### Inputs

| Input | Width and interpretation |
|---|---|
| Containing-object address | A 32-bit address value. The embedded record begins at address plus `0x18`. |
| Selector | An unsigned 32-bit operation identifier. Recognized values are `0x2D01` through `0x2D06`. |
| Adjustment | An unsigned 32-bit bit pattern. Arithmetic wraps at 32 bits; particular paths subsequently retain eight or sixteen bits or interpret a result as signed. |
| Auxiliary value | An uninterpreted 32-bit value stored in temporary storage on time paths. No explicit onward argument or result dependence is established. |

There is no visible validation of address, alignment, memory extent, field validity, or adjustment range.

Calendar paths directly read twelve bytes through three 32-bit accesses beginning at object offset `0x18`. Time paths directly read one byte at object offset `0x1A` and one at `0x19`. Dependency-specific memory requirements are unknown.

The unsupported-selector path performs no visible object dereference and invokes no dependency. A dereferenceable object is therefore not required by that visible path.

### Output

The result is a packed 64-bit value:

| Result bits | Meaning |
|---|---|
| 0–31 | Completion indicator, either zero or one. |
| 32–63 | Original selector on unsupported and time paths; first temporary-record word on calendar paths. |

On calendar paths, the payload is the first 32-bit word as used when assembling the return value. It is not established to be an error code, a comparison result, or a single date field. Dependencies receive the temporary record’s address, so their possible mutations must not be excluded from the payload’s meaning.

Completion is set to one before notification. It does not certify conversion validity, successful persistence, or any undocumented dependency postcondition. A packed result is available to the caller only if execution returns normally.

## Behavior

### Record representation and byte-order limits

Calendar paths load three 32-bit words from successive record addresses:

| Word | Address relative to record | Direct interpretation |
|---|---|---|
| First | `0x00` | Bits 24–31 are the day-like field; direct day edits preserve bits 0–23. |
| Second | `0x04` | Bits 0–7 are the month-like field; bits 16–31 are the year-like field; bits 8–15 have no established meaning. |
| Third | `0x08` | Snapshotted without direct field interpretation. |

Direct month edits preserve bits 8–31 of the second word. Direct year edits preserve bits 0–15 of that word. These are word-relative guarantees, not unconditional memory-byte mappings.

**Conditional little-endian interpretation only:** if the loaded words use little-endian storage, the day-like field occupies record byte `0x03`, the month-like field occupies byte `0x04`, and the year-like field occupies bytes `0x06`–`0x07`. Under that same assumption, byte `0x05` corresponds to second-word bits 8–15 and is preserved by both direct month and direct year edits.

No independent byte-order evidence is supplied. Without the little-endian assumption, those packed fields must not be assigned those byte offsets.

Separately, time paths explicitly access record byte `0x02` as the hour-like input and byte `0x01` as the minute-like input. Those byte addresses are established independently of the packed-word interpretation. Under little-endian storage they correspond to first-word bits 16–23 and 8–15, respectively; that correspondence is conditional.

### Dispatch

| Selector | Operation |
|---|---|
| `0x2D01` | Add adjustment × 3,600 to the time subtotal. |
| `0x2D02` | Add adjustment × 60 to the time subtotal. |
| `0x2D03` | Add adjustment × an unknown stored multiplier × 64 to the time subtotal. |
| `0x2D04` | Adjust the day-like field. |
| `0x2D05` | Adjust the month-like field and conditionally reduce the day-like field. |
| `0x2D06` | Adjust and bound the year-like field. |

Every other selector returns completion zero with the original selector as the high result word, without visible object access or dependency calls.

### Time paths

The function reads the two explicitly addressed bytes and forms the initial subtotal

\[
S = 3600H + 60M,
\]

where \(H\) and \(M\) are unsigned byte values. The conversion of the hour-like byte through a signed 16-bit representation does not make it negative: every possible byte value remains between zero and 255.

Both byte reads and the subtotal calculation precede F-b8274406a146. Their relative read order is not established by the supplied expression.

F-b8274406a146 receives the embedded-record address followed by the value `1`. Let \(B\) be the low 32 bits of its 64-bit result. The function retains the remainder

\[
R = B-S \pmod {2^{32}}.
\]

The dependency result’s high word is extracted, but does not contribute to the eventual conversion argument or return payload on any recognized time path.

The selected increment is added to the original subtotal:

- `0x2D01`: adjustment × 225 × 16, equivalent modulo \(2^{32}\) to adjustment × 3,600.
- `0x2D02`: adjustment × 60.
- `0x2D03`: adjustment × the stored multiplier × 64.

For `0x2D03`, the multiplier expression is evaluated after F-b8274406a146 returns. Its value, declared storage type, unit, and stability are unavailable. In particular, no preservation of that storage across the dependency call may be assumed.

All these arithmetic results are 32-bit machine values. Overflow wraps modulo \(2^{32}\). The adjusted subtotal is then interpreted as signed and corrected at most once:

- A negative value receives an addition of 86,400.
- A nonnegative value of at least 86,400 receives a subtraction of 86,400.
- A value from zero through 86,399 remains unchanged.

This is not full reduction modulo 86,400. The corrected value can remain negative or at least 86,400.

F-dee541b2a059 then receives the 32-bit sum of \(R\) and the corrected subtotal, followed by the embedded-record address. Its return value is ignored.

Completion becomes one, and F-3d31f2719b34 receives the containing-object address. If notification returns normally, the high result word remains the original selector.

The initial subtotal is not recomputed after F-b8274406a146. Any record mutation by that dependency does not replace the earlier byte values in this arithmetic.

### Calendar snapshot

For `0x2D04`–`0x2D06`, the function reads the three 32-bit record words in ascending address order and places them in a twelve-byte temporary record.

No atomic snapshot, locking, or concurrent-update policy is established. The direct edits described below modify temporary storage rather than the original object. Dependency mutations are additional possibilities, not excluded by these direct-edit guarantees.

### Day-like adjustment: `0x2D04`

F-8cfb43901820 first receives the temporary-record address. Its unsigned 32-bit result serves as a limit.

After that call, the function extracts the day-like field from bits 24–31 of the first temporary word. It adds the adjustment, retains the low eight bits, and installs that byte into the same bit range.

Bounding uses the retained byte:

- If it is zero, the field becomes one. The limit comparison is skipped.
- Otherwise, the full unsigned 32-bit dependency result is compared against it.
- If that result is smaller, the field becomes the result’s low eight bits.
- Otherwise, the retained byte remains.

The day input is read after the dependency call. Because preservation of temporary storage is not established, this need not be the original snapshot’s day.

A wrapped candidate of zero becomes one even when the dependency returns zero. A nonzero candidate can instead become zero when the dependency returns zero.

### Month-like adjustment: `0x2D05`

The function extracts second-word bits 0–7, adds the adjustment, and retains eight bits. It bounds that retained value:

- Zero becomes one.
- Values above twelve become twelve.
- Values one through twelve remain unchanged.

The direct edit replaces only second-word bits 0–7. There is no direct carry into the year-like field.

This is truncation followed by bounding, not saturation of the unbounded mathematical sum.

F-8cfb43901820 then receives the modified temporary record. After that call, the function reads the day-like field from first-word bits 24–31. If the dependency’s full unsigned 32-bit result is smaller, the day-like field becomes the result’s low eight bits.

Thus month bounding precedes the dependency call, while day limiting uses post-call storage. The dependency is not established to preserve the bounded month or any other temporary field.

### Year-like adjustment: `0x2D06`

The function extracts second-word bits 16–31, adds the adjustment, and retains the low sixteen bits as the candidate year.

The lower bound is 2001:

- If the candidate is below 2001, second-word bits 16–31 become 2001. The upper-bound expression is not evaluated.
- Otherwise, the upper-bound expression is evaluated for an unsigned comparison with the candidate.
- If the bound is smaller, a separate evaluation supplies the replacement, narrowed to sixteen bits.
- If the comparison is false, the candidate is retained.

The direct edit preserves second-word bits 0–15.

The comparison and replacement use separate source-level evaluations. This does not establish volatile behavior, two distinct physical memory reads, synchronization, or any mechanism allowing different observed values.

The upper bound’s value and declared type are unavailable. No coherent accepted interval is guaranteed without further constraints. For example, a stable upper bound below 2001 does not override the lower branch: a candidate below 2001 still becomes 2001, whereas a candidate at least 2001 can be replaced by the smaller upper bound.

This selector does not call F-8cfb43901820 and performs no direct day correction.

### Calendar comparison and conditional transfer

Following the selected edit, F-65fc54469e38 receives:

1. The original embedded-record address.
2. The temporary-record address.

Its 32-bit result controls the remaining calls.

A zero result leaves completion zero and skips the explicit transfer and notification. A nonzero result causes a transfer-dependency invocation with the original-record address, temporary-record address, and byte count `10`, in that order. Completion then becomes one, and F-3d31f2719b34 receives the containing-object address.

The transfer dependency has no supplied opaque ID or implementation contract.

**Inferred intent:** comparison decides whether to commit a changed record, and transfer copies temporary bytes into the object. Only the argument construction and conditional invocation are established.

Under ordinary destination-first byte-copy semantics, the transfer would replace record bytes `0x00`–`0x09`. Bytes `0x0A`–`0x0B` were snapshotted but would lie outside that transfer. This byte-range observation does not depend on endianness.

A zero comparison result does not prove that the object is unchanged: preceding dependencies, including comparison itself, may have side effects.

## Side effects

The visible body directly reads object memory and writes temporary storage. It makes no direct object write. Object updates are delegated to dependencies; their actual effects remain unspecified.

Calendar dependencies receive an address into the temporary record. No explicit heap allocation or release appears. Passing that address does not establish ownership transfer, permission to retain it, or permission to release it. The temporary record is not a caller-owned output buffer and has no established usable lifetime after return.

Dependency preservation of globals, pointed-to storage, other memory, or ownership is unknown. An ignored return value does not imply absence of side effects.

There is no visible rollback, error translation, exception handling, or recovery. Invalid access, dependency failure, non-returning calls, and exceptional control transfer have no specified successful-return result.

## Dependencies

| Opaque ID | Established use |
|---|---|
| F-b8274406a146 | Receives embedded-record address and `1`; returns 64 bits. Its low word contributes to the time argument. |
| F-dee541b2a059 | Receives the computed 32-bit time-related value and embedded-record address. Return value ignored. |
| F-3d31f2719b34 | Receives containing-object address after time processing or an accepted calendar transfer. Return value ignored. |
| F-8cfb43901820 | Receives temporary-record address on day and month paths; returns the unsigned 32-bit limit used by those paths. |
| F-65fc54469e38 | Receives original-record address and temporary-record address; zero suppresses explicit transfer and notification, nonzero permits them. |

The three-argument transfer dependency’s identity and semantics remain **documented_only**. No opaque ID, copy guarantee, error convention, or ownership contract is invented for it.

## Acceptance examples

These are conditional static examples, not executed tests. Field values below refer to word-relative bit ranges unless an explicit byte address is stated.

1. **Unsupported selector.** Selector `0x12345678` returns `0x1234567800000000`, without a visible object read or dependency call.

2. **Explicit time-byte reads.** If record byte `0x02` yields 23 and byte `0x01` yields zero, the subtotal is 82,800 before F-b8274406a146. Either relative byte-read order is consistent with the evidence.

3. **Single forward correction.** With that subtotal, selector `0x2D01`, and adjustment two, the adjusted subtotal is 90,000 and the corrected subtotal is 3,600. If conversion returns low word \(B\), F-dee541b2a059 receives \(B-82{,}800+3{,}600\), modulo \(2^{32}\).

4. **Incomplete normalization.** From subtotal zero, `0x2D01` with adjustment 48 produces 172,800 before correction and 86,400 afterward, not zero.

5. **Signed backward correction.** From subtotal zero, `0x2D02` with adjustment `0xFFFFFFFF` produces the signed value −60 before correction and 86,340 afterward.

6. **Day wrap overrides a zero limit.** If post-dependency first-word bits 24–31 equal 255, adjustment one produces day one even when F-8cfb43901820 returns zero.

7. **Full-width day limit.** A post-dependency day of 30 plus adjustment one becomes 30 when the limit is 30. With limit 300, the candidate 31 is retained; the limit is not narrowed before comparison.

8. **Month truncation.** Second-word bits 0–7 equal to one, plus adjustment 255, produce zero after truncation and then one after bounding. A month of twelve plus one becomes twelve. These describe the direct edit before dependency mutation.

9. **Year wrap.** A year-like field of 65,535 plus adjustment one produces candidate zero and stored year 2001. The upper-bound expression is not evaluated.

10. **Conditional upper-bound evaluations.** Candidate 2200 with a stable bound of 2100 causes comparison and replacement evaluations and stores 2100. Candidate 2050 with the same bound requires only the comparison evaluation and retains 2050.

11. **Conditional byte mapping.** With little-endian word storage, direct month editing changes record byte `0x04`, and direct year editing changes bytes `0x06`–`0x07`; both preserve byte `0x05`. Without that storage assumption, acceptance must instead be checked against second-word bit ranges.

12. **Calendar gating.** A zero comparison result yields completion zero and no explicit transfer or notification. A nonzero result invokes transfer with count ten, then notification; if both return normally, completion is one. Neither case establishes undocumented dependency side effects.

## Uncertainties

The following remain **documented_only**:

- Storage byte order and the complete record schema.
- Valid calendar domain, including intended treatment of invalid starting fields.
- Time-multiplier value, declared type, unit, and stability.
- Year-bound value, declared type, storage semantics, and physical access behavior.
- Dependency mutation, aliasing, retention, allocation, release, and failure behavior.
- Comparison meaning beyond its zero/nonzero branch effect.
- Transfer identity and actual transfer semantics.
- Required alignment, synchronization, object lifetime, and dependency-specific memory extents.
- Intended public meaning of the high result word and auxiliary input.

Calendar, conversion, transfer, and notification terminology describes inferred roles, not independently verified callee semantics.

## Evidence

Established claims derive from the supplied static body and opaque dependency mapping: input widths, explicit byte addresses, word-load addresses and order, packed-word bit operations, arithmetic widths, conditional evaluations, call arguments, branch effects, and result assembly.

The body establishes edits to first-word bits 24–31, second-word bits 0–7, and second-word bits 16–31. It does not independently establish their memory-byte positions. The byte-offset interpretation of those packed fields is therefore explicitly conditional on little-endian storage.

No dependency implementations, global definitions, independent interface documentation, or runtime observations were supplied. No runtime tests have been performed.
