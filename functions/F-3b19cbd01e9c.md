<!-- Trusted provenance: F-3b19cbd01e9c; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-3b19cbd01e9c

## Purpose

F-3b19cbd01e9c incrementally assembles a value from a byte-oriented encoding using a caller-supplied input callback and state stored in the supplied object. Its prefix and continuation masks resemble UTF-8, but its established behavior is less restrictive than Unicode UTF-8 decoding.

The function:

- Reads input until it has assembled one value or encounters an error.
- Records intermediate progress in the supplied object.
- Stores a completed value and marks it as cached.
- Returns an already cached value without requesting more input.
- Returns −1 for input failure or a rejected encoding unit, with different state effects for those two cases.

It does not establish Unicode scalar validity. In particular, it does not reject overlong encodings, surrogate values, or assembled values above U+10FFFF.

The apparent intent is a stateful character decoder with cached results. That intent is inferred; the state transitions, masks, stores, and return behavior below are established by the supplied static evidence.

## Interface

### Input object and recovered layout

The sole argument is a pointer to mutable storage. The recovered interface uses 32-bit integer words and a 32-bit address representation:

| Byte offset | Role | Interpretation |
|---|---|---|
| +0 | Dispatch reference | Address of a dispatch structure containing the input callback |
| +4 | Decoder state | Signed 32-bit state selector |
| +8 | Accumulator or cached result | 32-bit stored value |

The input callback is obtained from the dispatch structure at byte offset +4 and is invoked with the original object pointer as its sole visible argument.

These offsets describe the recovered interface, not a portable host-language structure definition. The object may contain additional storage used by the callback; its size and layout beyond the three words are unknown.

The caller must provide accessible storage for the fields the selected execution path reads or writes. Any path requesting input additionally requires a valid dispatch reference and callable entry. There are no explicit null, bounds, alignment, or dispatch-validity checks.

### Return value

The return value is a signed 32-bit integer:

- On ordinary completion, it is the stored assembled value.
- In state 100, it is the stored value without further decoding.
- On detected input or encoding failure, it is −1.

With normal initialization and a callback that does not alter decoder fields, completed values are nonnegative. This is not guaranteed for an arbitrary preloaded accumulator or cached value. State 100 returns the stored bit pattern as a signed value without validation, so −1 is not an unambiguous error indicator for every possible supplied object.

### Callback result interpretation

The input callback, labeled **F-000000000001** in this contract, returns a 32-bit result:

- A result whose signed interpretation is negative indicates input failure.
- Every nonnegative result is processed by masks on its low bits.
- There is no explicit requirement or enforcement that a successful result lie between 0 and 255.

Consequently, nonnegative values above 255 can be accepted. Their higher bits do not contribute to the assembled payload.

## Behavior

### State meanings

| State | Operation on entry |
|---|---|
| 0 | Request and classify an initial unit |
| 1 | Request the final continuation of a two-unit sequence |
| 2 | Request the first continuation of a three-unit sequence |
| 3 | Request the final continuation of a three-unit sequence |
| 4 | Request the first continuation of a four-unit sequence |
| 5 | Request the second continuation of a four-unit sequence |
| 6 | Request the final continuation of a four-unit sequence |
| 100 | Return the cached value |
| Any other signed 32-bit value | Repeatedly inspect the state without an input request or an explicit state change |

The function does not return merely because an intermediate state has been stored. In ordinary execution, it continues decoding within the same invocation.

The accumulator must already contain the appropriate partial value when entering states 1 through 6. The function does not validate that relationship or reconstruct missing prefix information.

### State inspection and callback lookup

The decoder repeatedly reads the state field to select its next action. Whenever the selected action requests input, it obtains the callback through the object’s current dispatch reference.

The dispatch reference is not established as a once-per-invocation snapshot. A callback that changes it can affect a later input request in the same invocation.

The operation selected before a callback remains the operation used to interpret that callback’s result. A callback’s state-field changes do not retroactively change that selection. After the selected operation finishes, subsequent state inspection can observe callback changes that the decoder has not overwritten.

### Initial-unit processing: state 0

One input callback is made.

If its signed result is negative, the function immediately returns −1. This path performs no explicit write to the state or accumulator.

For a nonnegative result, classification uses low-bit masks:

| Condition | Meaning and explicit effects |
|---|---|
| Bit 7 is clear | Store the low seven bits as the completed value, then store state 100 |
| Bits 7–5 equal binary 110 | Replace the accumulator with the low five bits shifted left by 6, then store state 1 |
| Bits 7–4 equal binary 1110 | Replace the accumulator with the low four bits shifted left by 12, then store state 2 |
| Bits 7–3 equal binary 11110 | Replace the accumulator with the low three bits shifted left by 18, then store state 4 |
| None of these conditions holds | Return −1 without an explicit state or accumulator write |

After either completion or an accepted multibyte prefix, the function resumes state inspection.

For byte-valued callback results, the accepted initial ranges are 0x00–0x7F, 0xC0–0xDF, 0xE0–0xEF, and 0xF0–0xF7. A continuation byte cannot begin a sequence.

### Continuation processing

Every continuation request first checks for a negative signed callback result. Otherwise, the low byte must satisfy the continuation mask: bits 7–6 must equal binary 10.

A valid continuation contributes its low six bits. The placement depends on the state selected before the callback:

| Selected state | Accumulator operation | Following state |
|---|---|---|
| 1 | OR in the low six bits without shifting | 100 |
| 2 | OR in the low six bits shifted left by 6 | 3 |
| 3 | OR in the low six bits without shifting | 100 |
| 4 | OR in the low six bits shifted left by 12 | 5 |
| 5 | OR in the low six bits shifted left by 6 | 6 |
| 6 | OR in the low six bits without shifting | 100 |

For each accepted continuation, the accumulator is read after the callback returns. The resulting accumulator is stored before the following state is stored.

Thus, callback changes to the accumulator can affect the completed result. The function does not retain an independent protected copy of the partial value across an input callback.

These updates use bitwise OR rather than replacement of a masked field. Unexpected bits already present in the accumulator are retained.

### Completion and caching

On a completing path, the function stores the completed value, stores state 100, and returns to state inspection. When state 100 is observed, it reads and returns the stored value.

State 100 is not reset by this function. Repeated invocations in that state return the cached value without consuming further input. Starting another sequence requires an external state change; the mechanism responsible for that change is outside the supplied evidence.

### Error behavior

There are two materially different failure categories.

**Negative callback result during continuation processing:**

- Store state 0.
- Return −1.
- Do not explicitly alter the accumulator.

This applies to selected states 1 through 6. The existing partial accumulator, including callback changes, remains stored.

**Nonnegative callback result that fails the required prefix or continuation mask:**

- Return −1.
- Do not explicitly alter either state or accumulator after that callback.

With a callback that does not modify decoder fields, an invalid continuation therefore leaves the decoder waiting at the same continuation stage. A later call can continue from that stage using a later input unit. The rejected unit has already been requested; there is no explicit pushback operation.

A negative result requested from state 0 also returns −1 without an explicit state reset. Under a field-preserving callback, the state is already 0. If the callback changes the state, that distinction is observable.

No different error codes distinguish exhaustion, callback failure, invalid prefix, or invalid continuation.

### Arithmetic and encoding boundaries

All assembled contributions fit within 21 bits when decoding begins in state 0 and the callback does not change the accumulator:

- One-unit path: 0 through 0x7F.
- Two-unit path: 0 through 0x7FF.
- Three-unit path: 0 through 0xFFFF.
- Four-unit path: 0 through 0x1FFFFF.

The largest shift is 18 positions applied to a three-bit payload. These normal-path operations do not require arithmetic beyond 32 bits.

There are no checks for shortest-form encoding, surrogate exclusion, Unicode’s maximum scalar value, or special restrictions on early continuation units. A syntactically accepted sequence may therefore assemble a value that a strict UTF-8 decoder would reject.

### Unsupported states

For any state other than 0–6 or 100, the visible control flow repeatedly reads the state without making an input callback, returning, or repairing the object.

If the state remains unchanged, this is nontermination. The evidence does not establish synchronization semantics or justify relying on another thread to change the state and release the loop.

## Side effects

The function’s explicit writes are confined to the state and accumulator fields. It does not explicitly allocate storage, release storage, replace the dispatch reference, close an input source, or transfer ownership.

For successful state transitions, accumulator storage precedes state storage. On continuation input failure, state 0 is stored before the error return.

Callback effects are separate. F-000000000001 receives the mutable object pointer and may have additional externally visible behavior. The supplied evidence does not establish that it preserves:

- The decoder’s fields.
- The dispatch reference.
- Other object storage.
- Global state.
- Resource ownership or lifetime.

Ordinary decoding requires the object and any subsequently accessed dispatch storage to remain valid throughout the invocation. Whether the callback obeys that requirement is not established.

The function has no visible rollback. An input request and any callback effects remain observable even when the requested unit is rejected.

## Dependencies

### F-000000000001

This is a contract-local opaque label for the indirect input callback. It is not a resolved identity supplied by the dependency map.

Established interaction:

- Located through the object’s dispatch reference at dispatch offset +4.
- Called with the original object pointer.
- Supplies a 32-bit result.
- Negative signed results select input-failure behavior.
- Nonnegative results supply the low-bit encoding information.

Unknown behavior includes how input is obtained, whether requesting input advances a stream, what negative values mean individually, and whether the callback mutates fields or manages resources.

No other explicit function dependency is visible. The published identity of the subject itself is F-3b19cbd01e9c.

## Acceptance examples

Unless stated otherwise, these examples assume a valid object, state 0 on entry, and a callback that returns the listed values in order without changing decoder fields. They are conditional static expectations, not executed tests.

| Callback results or initial condition | Expected behavior |
|---|---|
| 0x41 | One callback; accumulator becomes 0x41; state becomes 100; return 65 |
| 0xC2, 0xA2 | Two callbacks; return 0xA2; final state 100 |
| 0xE2, 0x82, 0xAC | Three callbacks; return 0x20AC; final state 100 |
| 0xF0, 0x9F, 0x98, 0x80 | Four callbacks; return 0x1F600; final state 100 |
| 0xC0, 0x80 | Return 0 despite the overlong representation |
| 0xED, 0xA0, 0x80 | Return 0xD800; no surrogate rejection |
| 0xF7, 0xBF, 0xBF, 0xBF | Return 0x1FFFFF; no Unicode maximum check |
| 0x80 as the initial unit | Return −1; no explicit decoder-field write |
| 0xF8 as the initial unit | Return −1; no explicit decoder-field write |
| 0xC2, then 0x41 | Return −1 after two callbacks; retain state 1 and accumulator 0x80 |
| A subsequent invocation after the preceding failure supplies 0xA2 | Return 0xA2 and enter state 100 |
| 0xE2, then a negative result | Return −1; state becomes 0; accumulator remains 0x2000 |
| A negative result on the first request | Return −1 without an explicit state or accumulator write |
| State 100 with stored value 0x1234 | Return 0x1234 without a callback or explicit write |
| State 100 with stored bit pattern 0xFFFFFFFF | Return −1 without requesting input or detecting an encoding error |
| Initial callback result 0x141 | Return 0x41, because bit 7 is clear and only the low seven bits are retained |
| State 7 remains unchanged | No callback or return; repeated state inspection |

Callback mutation also has defined local consequences. For example, if a callback requested from state 1 returns 0x80 but first changes the accumulator to 0x4000, completion stores and returns 0x4000. The final continuation contributes zero, and the accumulator read occurs after the callback.

## Uncertainties

The following remain **documented_only**:

- The exact platform ABI and pointer representation beyond the recovered 32-bit layout.
- The callback’s concrete identity and complete calling contract.
- Whether callback results are restricted to bytes in actual use.
- The meaning of individual negative callback results.
- The object’s full size, initialization procedure, lifetime, and resource owner.
- Which component resets state 100 before another value is decoded.
- Whether callers intentionally enter intermediate states with preloaded accumulators.
- Whether callbacks can re-enter this function or mutate shared state.
- Any thread-safety, memory-ordering, signal-safety, or asynchronous progress guarantees.
- Whether accepting noncanonical or non-Unicode values is intentional.

These unknowns do not authorize assuming preserved globals, automatic resource release, strict UTF-8 validation, or recovery from unsupported states.

## Evidence

The supplied static function body establishes the state selectors, callback access, signed failure tests, bit masks, accumulator operations, store ordering, cached return path, and lack of a return path for unsupported unchanged states.

The UTF-8-like purpose is inferred from the prefix patterns and six-bit continuation assembly. Strict Unicode decoding is not established and is contradicted by the absence of the required range and shortest-form checks.

No callback implementation, caller implementation, ABI declaration, or runtime trace was supplied. No runtime tests have been performed.
