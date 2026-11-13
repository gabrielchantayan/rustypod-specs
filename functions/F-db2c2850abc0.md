<!-- Trusted provenance: F-db2c2850abc0; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-db2c2850abc0

## Purpose

F-db2c2850abc0 dispatches an event against an object. It performs mode-dependent processing, invokes direct and virtual dependencies, changes selected object storage, and returns a signed 32-bit result.

An entry-mode classification equal to `0x0800` selects specialized processing and a dedicated fallback. Other classifications select preliminary notification handling, flag-controlled updates, and general identifier matching.

A further mode-specific block follows a second classification check. With stable object memory, that block cannot be reached: the earlier check excluded the same mode, and no intervening call or explicit object write establishes a transition. Its conditional behavior remains part of the static evidence, without a claim of ordinary reachability.

Domain interpretations such as selection, refresh, or timer processing are inferred intent only.

## Interface

### Inputs and representation

The represented interface has three inputs:

1. **Object address:** identifies storage containing fields, embedded objects, and a virtual-operation table pointer.
2. **Event value:** an unsigned 32-bit address-bearing value identifying event storage.
3. **Callable-shaped argument:** its incoming value has no established behavioral influence. Paths that use its working storage first replace it.

Observed word accesses are 32 bits. Observed byte accesses are eight bits. Dependency results represented more widely are reduced to their low 32 bits where used as addresses or ordinary word values.

The following layouts describe accessed storage, not complete object definitions.

| Object offset | Observed use |
|---|---|
| `0x00` | Virtual-operation table pointer |
| `0x38` | Argument to F-5710a8628cba |
| `0x40` | Primary identifier |
| `0x44` | Primary event-group identifier |
| `0x48` | Flags: mode mask `0x1800`, suppression bit `0x02000000`, update bit `0x20000000` |
| `0x98` | Secondary event-group identifier |
| `0x9C` | Nonzero value enables secondary matching |
| `0xEC` | Pointer supplying another virtual-operation table |
| `0x100` | Signed index-like word |
| `0x104` | Word compared with a dependency result |
| `0x124` | Flags including `0x00000002` and `0x00010000` |
| `0x128` | Embedded-object address used for identity matching |
| `0x154` | Argument to F-4e95b66bff0e |
| `0x158` | Addressed byte and overlapping 32-bit field; distinct access widths matter |
| `0x1AC` | Second embedded-object address |
| `0x1E0` | Optional auxiliary-object pointer |

| Event offset | Observed use |
|---|---|
| `0x04` | 32-bit event kind |
| `0x08` | 32-bit identifier or value |
| `0x0C` | 32-bit identifier or value |
| `0x10` | Encoded 32-bit value for specialized kinds `6` and `7` |

No event access at offset `0x00` is established.

### Output and preconditions

The result is a signed 32-bit integer. Locally completed paths generally return `0` or `1`. Specialized auxiliary processing and specialized fallback can return an external result unchanged, including a non-Boolean result.

Each selected path requires valid readable storage, writable storage where specified, and callable virtual entries. No explicit main-object, event, extent, alignment, or virtual-entry validation is shown. Some dependency-produced addresses are dereferenced without null checks.

No ownership transfer, allocation obligation, or release obligation is established for inputs, embedded objects, or dependency-produced values.

## Behavior

### Entry reads and classification

There are **two separate, unconditional reads** of object offset `0x48` at entry:

1. The first supplies the value masked with `0x1800` for entry-mode classification.
2. The subsequent read supplies the flags retained for the specialized kind-`0x13` suppression decision.

These are not one shared snapshot. Their values need not agree if memory changes between reads.

Only an exact masked value of `0x0800` selects specialized processing. Masked values `0`, `0x1000`, and `0x1800` select the other path. The second read does not revise this classification.

### Specialized entry mode

Read the event kind at `0x04`. Recognized processing either returns early or reaches the specialized fallback; it does not continue into general processing.

#### Kind `0x13`

Check that object offset `0x1E0` is nonzero and that the retained **second entry flags read** has suppression bit `0x02000000` clear.

If eligible, call F-1ce14fb580a7 with the auxiliary pointer and event words at `0x0C` and `0x08`, in that argument order. The auxiliary pointer supplied to the call is read from the object; it is not contractually frozen by the preceding nonzero check.

If the dependency returns nonzero, call F-92bea7c30bd6 with the main object, then return the retained dependency result unchanged. Otherwise, continue to specialized fallback.

#### Kinds `6` and `7`

Read event offset `0x10` and test whether its top byte is zero. Only if that test fails, read `0x10` again and test bit `0x01000000`. Either successful test permits processing. The two tests must not be described as necessarily using the same snapshot.

If eligible:

1. Call F-11c1b7e655b3 with the event address; ignore its result.
2. Read the auxiliary pointer at object offset `0x1E0`.
3. If nonzero, reread event `0x10` and extract its low 16 bits.
4. Compare that unsigned value inclusively with unsigned byte bounds at auxiliary offsets `0x08` and `0x09`. The upper-bound check is conditional on passing the lower-bound check.

If within bounds, enumerate signed indices starting at zero. Before each iteration, reread the auxiliary pointer, follow its pointer at `0x04`, and compare the index with the signed 32-bit count at that referenced object's `0x04`. A nonpositive initial count produces no iterations. Neither auxiliary storage nor count is an immutable snapshot.

For each admitted index:

- Call F-71f95996d3e8 with the then-read auxiliary pointer and index.
- Use the result's low 32 bits as an entry address.
- Skip null entries.
- For a nonnull entry, test bit `0x04` of its byte at `0x04`.
- If set, invoke its virtual operation at table offset `0x40`, passing the entry and a fresh full-width read of event `0x10`.

A nonzero entry-operation result marks the event handled but does not terminate enumeration.

After each successful entry operation, read a separate global control byte. If zero, skip both main-object operations below. If nonzero:

1. Invoke main-object virtual operation `0x174` with the index.
2. Reread flags at object `0x124`.
3. If bit `0x00010000` is clear, invoke main-object virtual operation `0x17C` with that index.

After enumeration, return `1` if any entry operation succeeded. Otherwise, continue to specialized fallback.

#### Kind `0x0D`

Call F-11c1b7e655b3 with the event address. Only a result exactly equal to `0x6F` permits main-object virtual operation `0x188`.

If that operation returns nonzero:

1. Read the object pointer at main-object `0xEC`.
2. Invoke the pointed-to object's virtual operation `0x170`, with no explicit arguments established.
3. Pass its 32-bit result to main-object virtual operation `0x17C`.
4. Return `1`.

Otherwise, continue to specialized fallback.

#### Specialized fallback

Call F-4956308dd593 with the main object and event value. Return its result unchanged.

### Other entry modes

Read event kind `0x04`, initialize a local handled result to zero, then reread object mode bits at `0x48`.

If this later mode value is not `0x0800`, only kind `0x0F` receives preliminary processing:

1. Call F-682adfd5a561 with the event value and selector `0x0D00`.
2. Use its low 32-bit result as an address and read the pointer at result offset `0x08`.
3. Compare that pointer with the main object's embedded address at `0x128`.
4. On equality, invoke main-object virtual operation `0x1BC` with value `1`.
5. After the call, store zero to the addressed byte at object `0x158`.
6. Set the local handled result to `1`.

Only that byte is explicitly cleared. Processing continues to flag-controlled updates rather than returning immediately.

### Conditional second-check mode block

This block requires the later mode read to equal `0x0800`, despite the initial classification being unequal. No intervening call or explicit write supplies that change.

If reached:

- Kinds `6`, `7`, and `0x0D` call F-682adfd5a561 with selector `0x0A00`, then pass its low 32-bit result to virtual operation `0x1B4`.
- Kind `0x15` uses selector `0x3500`, then virtual operation `0x1B8`.
- Those virtual results become the local handled result without immediate Boolean normalization.
- Kind `0x0F` uses selector `0x0D00` and reads a pointer at the extracted address's `0x08`.

For a kind-`0x0F` pointer matching embedded address `0x128`:

1. Read the **addressed byte** at object `0x158` and test bit zero.
2. Invoke virtual operation `0x1BC` with `1` if that bit was set, otherwise `0`.
3. After the call, read the **32-bit field** at object `0x158`.
4. Convert that word to a character-width value, subtract one, and assign the result to a character-width value.
5. Store that resulting character at the addressed byte `0x158`.
6. Branch on whether the resulting character is zero.

These are distinct byte and word accesses. The post-call operation is not established as an independent reread and decrement of the same addressed byte. Equating the word's low-order byte with the byte at the lowest address requires little-endian storage; character conversion behavior also matters.

For a zero resulting character, perform this ordered chain:

- F-a66a9be680df with no explicit arguments.
- F-c5a232e4455b with that result and `0x0C`.
- F-4e95b66bff0e with the next result, zero, and the then-read object word at `0x154`.
- F-8e50f265e4fa with the main object and final result.

For a nonzero resulting character, call F-ec4cfa552cce with embedded address `0x128`. Either outcome sets the local handled result to `1`.

If the extracted pointer instead matches embedded address `0x1AC`, call F-fdd8589eca2d, then F-bfa7e30ab583. If the latter returns zero, invoke virtual operation `0x1D8` with an unresolved global value. Otherwise, call F-ec4cfa552cce with embedded address `0x1AC`. This branch does not itself set the handled result.

### Flag-controlled updates

After preliminary processing, reread object flags `0x48`. If bit `0x20000000` is set, reread the event kind. For kind `0x13`, compare event `0x0C` with object `0x44`.

On a match, form a 32-bit signed-looking difference between event `0x08` and an unresolved global base. Process these cases in order.

**Difference zero:**

1. Call F-5710a8628cba with object word `0x38`; retain its signed 32-bit result.
2. Establish a later-operation condition, initially true.
3. If object flag `0x00000002` at `0x124` is set, invoke virtual operation `0x188`. A nonzero result makes the condition false and skips the following equality test.
4. Otherwise, compare object `0x104` with the retained result; equality makes the condition false.
5. Call F-b3f4a7a52609 with the object.
6. If the retained result is nonzero and no greater than this dependency's signed result, perform an index update:
   - If signed object `0x100` is less than the retained result, invoke `0x190` with the retained result minus one, then `0x174` with that same retained arithmetic value.
   - Otherwise, invoke `0x190` with object `0x100` and an additional value `1`; afterward reread `0x100` and pass it to `0x174`.
7. Invoke `0x16C` with the retained result.
8. If the previously established condition remains true, invoke `0x174` with the then-current object `0x100`.
9. Call F-92bea7c30bd6, then F-e6019f8ee60b, both with the object.
10. Return `1`.

**Difference `0x7F6`:**

Call F-5710a8628cba with object `0x38` and a second unresolved global value. Pass its result to virtual operation `0x168`, call F-e6019f8ee60b with the object, and return `1`.

**Otherwise, difference plus another unresolved global value equals zero:**

Call F-92bea7c30bd6 with the object and return `1`.

Other matching-event differences go directly to general fallback, bypassing the local handled-result check.

When the update flag is clear, or its kind/group match fails, return `1` if the local handled result is nonzero. Otherwise continue to general fallback.

### General fallback

Reread event kind `0x04`. If not `0x13`, return `0`.

Read event `0x0C` and retain it. If it equals object `0x44`:

1. Read event `0x08` and object `0x40`.
2. If they are equal, invoke virtual operation `0xD4` and return `1` **without reading the alternative global identifier**.
3. Only if they differ, read that alternative global identifier. If it equals the already-read event value, invoke `0xD4` and return `1`.

If primary matching does not accept, read object `0x9C`. When nonzero, compare the retained event `0x0C` with object `0x98`. Only on equality, reread event `0x08` and read a separate secondary global identifier.

If these secondary identifiers match, call F-5710a8628cba with no explicit arguments, retain only its low byte, pass that byte to virtual operation `0xA0`, and return `1`.

Otherwise return `0`.

## Side effects

Explicit object writes are limited to the addressed byte at `0x158`: a zero store or the character-width result of the conditional word-to-character decrement operation. No explicit write to the other bytes of that word is shown.

Calls may have additional effects. Nothing establishes that dependencies preserve object fields, event storage, globals, virtual tables, or referenced objects. Reads following calls remain subsequent observations, not reusable earlier snapshots.

There is no explicit allocation, release, logging, rollback, or exception handling. Invalid storage is not converted into a defined error result. Return values are interpreted only at the branches described above.

Signed comparisons govern enumeration counts and index-update comparisons. Word arithmetic must not be silently replaced by unbounded arithmetic. Overflow and out-of-range character conversions remain unresolved where the representation does not establish their target semantics.

## Dependencies

Direct dependencies are identified only by opaque IDs.

| Dependency | Established use |
|---|---|
| F-1ce14fb580a7 | Auxiliary event processing; nonzero result propagated |
| F-4956308dd593 | Specialized fallback; result propagated |
| F-11c1b7e655b3 | Event processing; exact result test on kind `0x0D` |
| F-71f95996d3e8 | Indexed entry retrieval |
| F-682adfd5a561 | Selector-based extraction |
| F-92bea7c30bd6 | Object operation on several completed paths |
| F-5710a8628cba | Word or byte-producing operation with differing represented argument lists |
| F-b3f4a7a52609 | Signed comparison limit |
| F-e6019f8ee60b | Final object operation on two update paths |
| F-a66a9be680df, F-c5a232e4455b, F-4e95b66bff0e, F-8e50f265e4fa | Ordered zero-character chain |
| F-ec4cfa552cce | Receives an embedded-object address |
| F-fdd8589eca2d, F-bfa7e30ab583 | Ordered second-embedded-object processing |

F-147f81afa77f identifies the shared general-fallback region; a distinct external invocation is not established.

Indirect dependencies are identified by virtual-table byte offsets because target function IDs are unavailable. Their complete signatures, ownership rules, and effects remain unknown.

## Acceptance examples

These are conditional static criteria, not executed tests.

1. **Independent entry reads:** If the first flags read classifies as `0x0800` and the second contains `0x02000000`, specialized kind-`0x13` auxiliary processing is suppressed. The first read still controls entry classification.

2. **Exact classification:** An initial masked mode of `0x1800` does not select specialized processing merely because bit `0x0800` is present.

3. **Non-Boolean propagation:** Eligible specialized kind-`0x13` processing returning `7` calls F-92bea7c30bd6 and returns `7`.

4. **Repeated encoded-value reads:** If the first eligibility read has a nonzero top byte but the conditional second read has bit `0x01000000` set, processing is eligible. Stable top byte `0x02` rejects; stable top byte `0x03` accepts.

5. **Enumeration continues:** Given two qualifying entries that both succeed, both operations occur in index order. A first success does not terminate enumeration. Assuming enumeration terminates, the final result is `1`.

6. **Global-byte gating:** A zero enumeration-control byte skips both `0x174` and `0x17C`. If nonzero and `0x174` sets flag `0x00010000`, the subsequent `0x17C` is skipped.

7. **Byte-only clearing:** A nonspecial kind-`0x0F` match at embedded `0x128` invokes `0x1BC` with `1`, then clears only addressed byte `0x158`, assuming the dependency returns.

8. **Distinct counter accesses:** If the conditional block is reached and the post-call 32-bit field at `0x158` is `1`, conversion and subtraction produce zero and select the zero-character chain, irrespective of the earlier addressed-byte bit test. Under little-endian, truncating eight-bit character storage, a post-call word of zero instead stores `0xFF` and selects F-ec4cfa552cce.

9. **Conditional primary global read:** A direct event-identifier/object-identifier match invokes `0xD4` without reading the alternative global. A mismatch reads that global before deciding whether the alternative accepts.

10. **Low-byte result:** A successful secondary fallback with F-5710a8628cba returning `0x123456AB` passes byte value `0xAB` to `0xA0`.

## Uncertainties

- **documented_only:** Values, storage details, and mutability of unresolved globals, including the enumeration-control byte.
- **documented_only:** Whether asynchronous mutation can reach the second-check mode block; no supported concurrency or synchronization contract is established.
- **documented_only:** Endianness, character signedness, and out-of-range character conversion rules. These affect interpreting overlapping byte/word accesses as one counter.
- **documented_only:** Signed overflow semantics for word subtraction, addition, and index-related arithmetic.
- **documented_only:** Complete ABI and calling conventions, especially the differing argument lists for F-5710a8628cba and virtual operation `0x190`.
- **documented_only:** Meaning of unused upper result words.
- **documented_only:** Dependency ownership, destruction, reentrancy, and preservation guarantees.
- **Inferred intent only:** The object appears to coordinate indexed entries and notifications; that interpretation adds no domain-specific contract.

## Evidence

The basis is the supplied static representation: access widths, offsets, conditional reads, branch predicates, call ordering, explicit writes, and return propagation.

In particular, it establishes two separate entry flag reads; a conditional counter path with an initial addressed-byte read followed by a post-call 32-bit read and character conversion; and an alternative primary identifier global read performed only after direct identifier mismatch.

Apparent types and signatures are not independent ABI validation. No dependency implementations, runtime tests, execution traces, or independent layout validation were available or performed.
