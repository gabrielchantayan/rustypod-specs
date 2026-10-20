<!-- Trusted provenance: F-382ffcbbd77a; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-382ffcbbd77a

## Purpose

F-382ffcbbd77a selects a value from an unsigned request and a shared limit, then writes a short-width representation of that value into a destination field at byte offset 12 from a supplied base address.

The selection has two distinct regimes:

- Requests from 0 through 128, inclusive, select 128 without consulting the shared limit.
- Requests of 129 or greater consult the shared limit. A request below that limit selects the request; a request equal to or above the limit selects a subsequently read limit value.

This is not an unconditional clamp between a minimum and maximum. In particular, the low-request regime ignores the shared limit even when that limit is below 128.

**Established static evidence:** the supplied body contains this selection and a single destination assignment on each returning path. It contains no calls.

**Inferred intent:** the destination may represent a bounded quantity with a nominal minimum of 128. Neither the quantity’s meaning nor a valid range for the shared limit is established.

## Interface

### Inputs

The function accepts two arguments:

| Input | Role | Interpretation |
|---|---|---|
| Destination base | Identifies memory containing the destination field | Supplied through an integer-typed argument and used to form an address |
| Requested value | Controls selection | Unsigned integer; compared against 129 and, conditionally, the shared limit |

The destination address is the base plus 12 bytes. The function writes a short-width object there. It does not read the existing destination value.

### Widths and representation

The supplied representation uses integer and unsigned-integer arguments and a short-width destination. The operational interpretation used for the concrete examples is:

- 32-bit destination-base representation;
- 32-bit unsigned request;
- 16-bit destination write.

These widths are consistent with the supplied representation, but the excerpt alone does not independently establish the target ABI. Accordingly, the 32-bit and 16-bit interpretation is an explicit assumption for numerical acceptance examples, not a portable guarantee derived solely from the type spellings.

Under that interpretation, the destination occupies bytes 12 and 13 relative to the supplied base. The selected value’s low 16 bits are stored. A reader interpreting that field as signed may observe a negative value when bit 15 is set. Byte order is not established.

The shared limit’s declared type and storage width are absent from the excerpt. Its value participates in a comparison with the unsigned request and, on one path, is converted to the destination’s short-width representation.

### Output

There is no return value. The observable result is the destination write, together with any shared-limit reads required by the selected path.

No status code distinguishes a normal write from an invalid-address failure. No exception, fault, or recovery mechanism is shown.

### Preconditions

For defined, successful execution:

- Forming the destination address must be valid in the execution environment.
- The resulting destination location must permit a short-width write.
- Any alignment requirements imposed by that environment must be satisfied.
- When the request is at least 129, the shared limit must be readable.

The function performs no visible validation of these conditions. It does not establish the size or validity of an enclosing object.

## Behavior

### Requests from 0 through 128

For every request below 129:

1. The selected value is 128.
2. The shared limit is not read.
3. The function writes the short-width representation of 128 at destination offset 12.
4. The function returns.

Both endpoints belong to this regime: requests of 0 and 128 produce the same stored value.

The shared limit cannot lower or raise the result on this path. Its accessibility is also irrelevant to this path, provided evaluating the request and writing the destination are otherwise valid.

### Requests of 129 or greater

For a request of at least 129, the function reads the shared limit for comparison.

If the comparison determines that the limit is greater than the request:

- The request is selected.
- The function writes its short-width representation at destination offset 12.
- The function returns.

If the comparison determines that the limit is less than or equal to the request:

- The function evaluates the shared limit again for the value to be stored.
- It writes that second evaluation’s short-width representation at destination offset 12.
- It returns.

Equality therefore takes the limit-selected path. With a stable limit, equality produces the same numerical result that selecting the request would produce. The distinction matters if reading the limit has observable effects or its value can change.

### Read and write ordering

The source-level access sequence is:

| Condition | Shared-limit evaluations | Destination writes |
|---|---:|---:|
| Request below 129 | None | One, selecting 128 |
| Request at least 129 and below the first limit value | One | One, selecting the request |
| Request at least 129 and at least the first limit value | Two | One, selecting the second limit value |

On the last path, the comparison evaluation precedes the value evaluation, and both precede the destination write.

This describes the evaluations expressed by the supplied body. The excerpt does not establish volatility, atomicity, synchronization, or the exact number of hardware load instructions. Those properties must not be inferred from this table.

There is no destination read, no retry, and no post-write verification.

### Stable-limit interpretation

When the shared limit remains stable and has the expected unsigned interpretation, requests of at least 129 select the smaller of the request and the limit.

That description must not be extended to requests below 129. For example, a stable limit of 64 does not prevent a request of 100 from storing 128.

Nor does this selection guarantee that the stored field preserves the selected numerical value. Narrowing occurs after selection. A selected value outside the destination’s representable range may produce a different signed interpretation or, under the assumed 16-bit machine conversion, retain only its low 16 bits.

### Changing limit values

The function does not explicitly preserve the comparison-time limit value for the limit-selected write.

At the source-evaluation level, if the first limit evaluation allows the limit-selected path and the second yields a different value, the second value supplies the destination data. There is no second comparison against the request.

This observation is not a concurrency guarantee. Whether external modification between evaluations is defined, visible, or possible depends on declarations and execution rules absent from the excerpt.

## Side effects

The only explicit write is to the short-width field at destination offset 12.

No allocation, release, ownership transfer, callback, logging, or external function invocation appears in the supplied body. The destination base remains caller-provided storage; the function neither acquires ownership nor releases it.

No explicit assignment targets the shared limit. However, non-aliasing is not established. If the destination overlaps the shared limit’s storage, the destination write may also modify that storage. Any shared-limit evaluations on the selected path occur before the destination write.

The body does not explicitly modify neighboring fields. This is not evidence about fault behavior, memory-mapped devices, or other effects of the underlying memory system.

The write is not established to be atomic. No locking or memory-ordering operation is shown.

## Dependencies

There are no function dependencies in the supplied body, so no additional opaque function IDs apply.

The function depends on these data and execution facilities:

- Caller-supplied storage addressable at base plus 12 bytes.
- A shared limit consulted only for requests of at least 129.
- Integer comparison and short-width conversion rules.
- The execution environment’s address formation, alignment, and memory-access behavior.

The shared limit’s initialization, permitted range, mutability, and ownership are unknown. No other function’s behavior is needed to explain the visible body, and no preservation or memory-management guarantees are assumed from unspecified callees.

## Acceptance examples

These are conditional behavioral examples derived from static evidence, not executed tests. Unless stated otherwise, they assume valid writable destination storage, a stable shared limit, a 32-bit unsigned request, a 16-bit destination, and ordinary low-16-bit narrowing.

| Request | Shared limit | Expected destination field |
|---:|---:|---|
| 0 | 1,000 | 128; limit is not evaluated |
| 128 | 1,000 | 128; limit is not evaluated |
| 129 | 1,000 | 129 |
| 999 | 1,000 | 999 |
| 1,000 | 1,000 | 1,000; limit-selected path |
| 1,001 | 1,000 | 1,000 |
| 100 | 64 | 128; low-request rule bypasses the limit |
| 129 | 64 | 64 |
| 129 | 0 | 0 |

Additional boundary and representation examples:

- **Narrowing the selected request:** with request 65,536 and limit 70,000, selection chooses the request. Under 16-bit narrowing, the stored bit pattern is zero.
- **Narrowing the selected limit:** with request 70,000 and limit 65,536, selection chooses the limit. Under the same assumption, the stored bit pattern is zero.
- **Signed interpretation:** with request 65,535 and limit 70,000, the stored pattern is all 16 bits set. An unsigned reader sees 65,535; a two’s-complement signed 16-bit reader sees −1.
- **Distinct limit evaluations:** with request 300, a first limit value of 200 selects the limit path. If the second evaluation can validly yield 400, the selected write value is 400, not 200 or 300. This example requires an environment in which those distinct evaluations and the change are defined and observable.
- **Invalid destination:** no successful result is promised when the destination field is unwritable or improperly aligned. The function contains no rejection path or fallback.
- **Untouched prior value:** the destination’s initial contents do not influence selection. Successful execution replaces the field rather than combining with its old value.

## Uncertainties

The following are **documented_only**:

- **Target ABI:** exact widths, alignment, address representation, and byte order are not independently supplied.
- **Shared-limit type:** signedness and width are missing. Comparison behavior for unusual bit patterns depends on that declaration and the language’s conversion rules.
- **Out-of-range short conversion:** low-bit truncation is the assumed machine interpretation used in examples. Portable language-level behavior for conversion to a signed short outside its range is not established here.
- **Address arithmetic:** wrapping, overflow, and invalid-address behavior cannot be determined from the excerpt.
- **Concurrent access:** no synchronization contract or atomicity guarantee is visible.
- **Hardware reads:** two source-level limit evaluations do not establish two physical loads.
- **Aliasing:** no constraint excludes overlap between the destination and shared-limit storage.
- **Semantic units:** the destination field’s purpose and units are unknown.

No validation, error policy, stronger bound, or ownership rule is inferred to fill these gaps.

## Evidence

The supplied static body establishes:

- An unsigned-request threshold separating values below 129 from all others.
- Selection of 128 on the low-request path.
- Conditional comparison against a shared limit.
- A separate evaluation of that limit on the limit-selected path.
- A short-width write at byte offset 12.
- No return value and no function calls.

The bounded-quantity interpretation is inferred intent, not an established application-level requirement. ABI details and environmental behavior remain conditional as identified above.

No runtime tests have been performed.
