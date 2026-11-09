<!-- Trusted provenance: F-0a4afe2aea73; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-0a4afe2aea73

## Purpose

F-0a4afe2aea73 conditionally issues six updates using three bytes obtained through a lookup dependency. It then constructs a temporary four-word record and independently checks whether to pass that record through additional dependencies. Its apparent 64-bit result contains the record’s final first and second words.

The updates resemble three two-digit time components, with a twelve-hour-like transformation applied to one component in one mode. This is inferred intent only. The supplied body establishes byte accesses, arithmetic-result usage, destination selection, and call ordering; it does not establish time encoding or visual destinations.

A null lookup result suppresses the six updates, not the record construction or subsequent eligibility check.

## Interface

### Arguments and result

The supplied representation exposes four arguments:

| Argument | Apparent width | Established use |
|---|---:|---|
| Context | 32 bits | Address used for field reads and interior-address calculations. |
| Second argument | 32 bits | No explicit use in the supplied body. |
| Third argument | 32 bits | No explicit use in the supplied body. |
| Fourth argument | 32 bits | Copied unchanged into the temporary record’s fourth word. |

The apparent 64-bit result is assembled from the final record contents:

- Low 32 bits: first record word.
- High 32 bits: second record word.

Those words initially contain the context address and that address plus `0x80`. Dependencies receiving the record address may change them before return.

The unused arguments’ widths and lack of explicit use do not establish additional calling-convention guarantees.

### Context layout

A word below is 32 bits. Offsets are relative to the original context address.

| Offset | Access | Role |
|---|---|---|
| `0x34` | Word read | Optional related-object value, captured only when second-phase processing is eligible. |
| `0x38` | Word read | First lookup argument. |
| `0x44` | Word read | Captured before lookup and later copied into the record. |
| `0x80` | Address calculation | Context-interior address stored in the record and passed to a dependency. |
| `0xA0` | Byte read | Nonzero suppresses second-phase processing. |
| `0xE8` | Word read | First update destination. |
| `0xEC` | Word read | Second update destination. |
| `0xF4` | Word read | Third update destination. |
| `0xF8` | Word read | Fourth update destination. |
| `0x100` | Word read | Fifth update destination. |
| `0x104` | Word read | Sixth update destination. |
| `0x108` | Word read | Additive base for update values, except the explicitly suppressed first value. |
| `0x10C` | Byte read | Zero selects transformed processing; nonzero selects direct processing. |

Destination words are passed as values to F-8ef069adf52b. Their concrete types and validity requirements are not established.

### External data

Two external-data operations are distinct:

1. An external lookup operand is evaluated as the second argument to F-5710a8628cba. Its declared width, contents, interpretation, and mutability are not established.
2. An external pointer is evaluated late in the eligibility check, and one byte is read through that pointer. This is an indirect read, not merely a read of a directly identified external byte.

The eligibility pointer has an apparent 32-bit address representation. No visible null check precedes its dereference.

### Preconditions and lifetime

The context is not checked for null. Every context field reached on the selected path must be readable at the time of access.

A nonnull lookup result must permit the reached byte reads at offsets 2, 1, and 0. Offsets 1 and 0 are each read twice, with dependency calls between the reads.

The temporary record comprises four consecutive 32-bit words, totaling 16 bytes. Its address is passed to dependencies, but the supplied body establishes neither ownership transfer nor a valid lifetime beyond this invocation.

Dependencies may impose additional preconditions. Their implementations are not supplied.

## Behavior

### Initial capture and lookup

The first explicit field read captures the word at context offset `0x44`.

The function then invokes F-5710a8628cba with:

1. The word at context offset `0x38`.
2. The evaluated external lookup operand.

Both argument evaluations occur before the lookup call. Their relative evaluation order is not established by the supplied representation.

A null lookup result bypasses all six updates. On that path, the update phase does not read the mode byte, update destinations, additive base, or lookup bytes.

A nonnull result enters the update phase.

### Arithmetic-result convention

F-d7be9cfb4000 receives a byte-derived value and the integer 10. The caller consumes two apparent 32-bit result channels:

- A primary channel for the first update in each pair.
- A secondary channel for the second update in each pair.

Some call sites expose both channels together; others expose the primary result or consume a secondary result from the same call. The supplied body supports this two-channel usage but does not establish the dependency’s arithmetic semantics.

Quotient and remainder are plausible interpretations, not unconditional requirements. Numerical examples below explicitly assume that interpretation.

Calls are not interchangeable with cached computations: repeated calls and intervening dependency effects are observable parts of the supplied behavior.

### First pair in zero mode

After a successful lookup, the function reads the byte at context offset `0x10C`. If it is zero, the function reads lookup byte 2 once and retains an unsigned eight-bit transformed value.

The transformation is:

- Subtract 12 once if the input exceeds 12.
- Replace a resulting zero with 12.

This is not general modulo-12 normalization. Inputs 0, 12, and 24 all produce 12; input 25 produces 13; input 255 produces 243. The selected subtraction cannot underflow, and the resulting value is within 1 through 243.

For a transformed value from 1 through 9:

- The first update value is exactly zero.
- No arithmetic call computes that value.
- The additive base is not read for that value.
- The destination at `0xE8` is read and passed to F-8ef069adf52b with zero.

For a transformed value of at least 10:

- F-d7be9cfb4000 is called with the retained value and 10.
- Its primary result is added to the word at `0x108`.
- The destination word at `0xE8` is then read.
- F-8ef069adf52b receives that destination and the computed sum.

After the first update returns, both zero-mode paths call F-d7be9cfb4000 with the retained transformed value and 10. Its secondary result is used for the second update.

Lookup byte 2 is not reread for this later call. Zero mode therefore makes one arithmetic call for this pair when the transformed value is 1–9, and two when it is at least 10.

### First pair in nonzero mode

If the mode byte is nonzero, lookup byte 2 is used directly.

The function calls F-d7be9cfb4000 once with that byte and 10, retaining both result channels. F-8ef069adf52b then receives:

- The destination at `0xE8`.
- The primary result plus the base at `0x108`.

The secondary result from this same arithmetic call is retained for the second update. There is no second arithmetic call for this pair.

Unlike the zero-mode single-digit path, this path includes the base even if the primary result is zero.

### Completion of the first pair

Both modes next invoke F-8ef069adf52b with:

- The destination at `0xEC`.
- The selected secondary result plus a fresh read of the base at `0x108`.

In nonzero mode, the secondary result was obtained before the first update. In zero mode, it comes from an arithmetic call after the first update.

The base and destination fields are not shared snapshots across the pair. Earlier dependencies may change them.

### Remaining pairs and total call counts

Lookup byte 1 is processed before lookup byte 0.

For byte 1:

1. The byte is read and passed with 10 to F-d7be9cfb4000.
2. The primary result plus the current base supplies the update to the destination at `0xF4`.
3. After that update returns, byte 1 is read again and passed with 10 to F-d7be9cfb4000.
4. This later call’s secondary result plus the current base supplies the update to the destination at `0xF8`.

Byte 0 follows the same pattern, using destinations at `0x100` and `0x104`.

For these updates, each destination and base is evaluated after the preceding arithmetic call and before the associated update call. Their relative order within the update call’s argument evaluation is not established.

With successful lookup and normal returns, the six update calls occur in destination-offset order:

`0xE8`, `0xEC`, `0xF4`, `0xF8`, `0x100`, `0x104`.

The total number of F-d7be9cfb4000 calls is:

| Path | Byte 2 calls | Byte 1 calls | Byte 0 calls | Total |
|---|---:|---:|---:|---:|
| Null lookup | 0 | 0 | 0 | 0 |
| Nonzero mode | 1 | 2 | 2 | **5** |
| Zero mode, transformed value 1–9 | 1 | 2 | 2 | **5** |
| Zero mode, transformed value at least 10 | 2 | 2 | 2 | **6** |

Thus, six arithmetic calls occur only on the last listed path, not on every path other than zero-mode single-digit processing.

### Record construction

After null lookup or completion of all updates, the function initializes the record in word order:

| Record offset | Initial value |
|---|---|
| `0x00` | Original context address. |
| `0x04` | Original context address plus `0x80`. |
| `0x08` | Word captured from context offset `0x44` before lookup. |
| `0x0C` | Fourth argument, unchanged. |

Changes to context offset `0x44` during lookup or updates do not refresh the third record word.

### Eligibility

After record initialization, the function calls F-3fac671648b8 with no explicit arguments.

The subsequent checks short-circuit in this order:

1. If that result is zero, skip eligible processing.
2. Otherwise, read the context byte at `0xA0`. If nonzero, skip eligible processing.
3. Otherwise, evaluate the external eligibility pointer and read its target byte. If zero, skip eligible processing.
4. A nonzero target byte permits processing.

A zero result from F-3fac671648b8 avoids both later reads. A nonzero context flag avoids evaluation and dereference of the external pointer.

The external pointer and its target byte are evaluated at this late point. No earlier snapshot substitutes for them; preceding dependencies may have changed either.

### Eligible processing and return

When eligible, the function reads and retains the word at context offset `0x34`. It then calls F-e57b05520b65 with:

1. The record address.
2. The original context address plus `0x80`.
3. That same context-interior address again.

If the captured related-object value is nonzero, F-ee8733a08210 receives the record address and the captured value plus `0xD0`.

The decision and address calculation use the capture, not a reread after F-e57b05520b65.

Next, F-ffe0d92c57b7 receives the original context address. If its result is nonzero:

1. F-c9b5d17d8d1e receives that result and the record address.
2. F-4d037ecf0f82 receives the same retained result and the record address.

The second consumer is not conditional on the first consumer’s return value. It can observe earlier record mutations.

Finally, the function returns the record’s first two words, first word low and second word high.

## Side effects

The explicit writes in the supplied body initialize the temporary record. No explicit write targets context fields, lookup storage, or either external datum.

Dependency effects are unknown. They may change context fields, lookup bytes, external state, or the record. No preservation, allocation, release, retention, or ownership-transfer guarantee follows from these call sites.

Updates are sequential, not transactional. No visible rollback reverses earlier updates after a later failure or abnormal return.

Null lookup, failed eligibility, and a zero result from F-ffe0d92c57b7 suppress work without a separate error status. Dependency return values other than the specifically consumed results do not visibly control error handling.

Arithmetic additions and derived addresses use apparent 32-bit operands. The representation does not independently settle source-language signed-overflow behavior, invalid-pointer behavior, alignment requirements, or recovery from faults. No saturation or overflow check is visible.

## Dependencies

| Opaque ID | Established call-site role |
|---|---|
| F-5710a8628cba | Lookup using a context word and an evaluated external operand; result checked for null. |
| F-d7be9cfb4000 | Arithmetic with a byte-derived value and 10; two result channels are consumed. |
| F-8ef069adf52b | Receives a destination word and an update value. |
| F-3fac671648b8 | Supplies the first eligibility result. |
| F-e57b05520b65 | Receives the record address and two identical context-interior addresses. |
| F-ee8733a08210 | Receives the record address and a captured related-object value plus `0xD0`. |
| F-ffe0d92c57b7 | Resolves a value from the original context address. |
| F-c9b5d17d8d1e | First consumer of the nonzero resolved value and record address. |
| F-4d037ecf0f82 | Second consumer of that same retained value and record address. |

These roles are not complete dependency contracts.

## Acceptance examples

These are conditional static acceptance cases, not executed tests. They assume reached memory accesses are valid, additions avoid overflow, and dependencies return normally.

### Null lookup and failed eligibility

If lookup returns null, neither arithmetic nor update calls occur. Record initialization and F-3fac671648b8 still occur.

If that eligibility result is zero, the function does not read the context flag or external eligibility pointer. The returned low word is the context address; the high word is that address plus `0x80`.

### Arithmetic values and counts

Assume F-d7be9cfb4000 supplies quotient and remainder, the base remains 100, and dependencies do not mutate relevant fields.

| Mode | Lookup byte 2 | Transformed value | First two update values | Total arithmetic calls |
|---|---:|---:|---|---:|
| Zero | 13 | 1 | 0, 101 | 5 |
| Zero | 0 | 12 | 101, 102 | 6 |
| Zero | 12 | 12 | 101, 102 | 6 |
| Zero | 24 | 12 | 101, 102 | 6 |
| Zero | 255 | 243 | 124, 103 | 6 |
| Nonzero | 7 | Not transformed | 100, 107 | 5 |

For zero mode with input 13, the first arithmetic call occurs after the first update. For nonzero mode with input 7, one arithmetic call supplies both channels before the first update.

### Repeated reads can yield a mixed pair

Under the same quotient/remainder assumption and a stable base of 100, suppose lookup byte 1 initially contains 42. Its first update receives 104.

If that update changes byte 1 to 57, the later reread makes the second update receive 107, not 102.

### Current external data is used

If the external lookup operand evaluates to different values on two invocations, the corresponding lookup calls receive those different values. No constant-value requirement is established.

If an earlier dependency changes the external eligibility pointer to a readable address containing a nonzero byte, that later pointer is used when both preceding eligibility checks pass. A cached byte from the former address would not match the established sequence.

### Captures and mutations

If lookup changes context offset `0x44`, the record’s third word still receives the pre-lookup capture.

If F-e57b05520b65 changes context offset `0x34`, the optional F-ee8733a08210 call still depends on the value captured before F-e57b05520b65.

If eligible dependencies leave the record’s first two words as A and B, the result contains A low and B high.

## Uncertainties

The following remain **documented_only**:

- Exact arithmetic semantics and lower-level result-channel convention of F-d7be9cfb4000.
- Semantic identities of the three input bytes, update destinations, and related objects.
- Declared width, contents, interpretation, and mutability of the external lookup operand.
- Contents and mutability of the external eligibility pointer; identity, lifetime, readability, and mutability of its target.
- Dependency effects on globals, context fields, lookup storage, record contents, and resource ownership.
- Lower-level calling convention, alignment, and evaluation ordering not established above.
- Overflow behavior, invalid-address behavior, abnormal dependency returns, and consequences of retaining the temporary record address beyond its lifetime.

No safe fallback is established when an otherwise reached access is invalid.

## Evidence

The supplied body statically establishes the explicit field offsets, byte and word accesses, branch conditions, dependency-call sequence, repeated byte reads, four-word record layout, and final word composition.

The arithmetic-call totals follow directly from the branches: nonzero mode has one call for byte 2; zero mode has one or two; bytes 1 and 0 each have two. Successful lookup therefore produces five or six arithmetic calls under the conditions specified above.

Time-component intent and quotient/remainder semantics are inferences. Dependency implementations and broader ownership contracts were not supplied. No runtime tests have been performed.
