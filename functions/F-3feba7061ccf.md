<!-- Trusted provenance: F-3feba7061ccf; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-3feba7061ccf

## Purpose

F-3feba7061ccf mediates delivery of a byte sequence through a context-selected callback, with an alternate path that stages data in an internal buffer.

The established behavior is a combination of:

- A mandatory signed-positive input-count check.
- A route selection based on existing buffered state and, when that state is zero, a bitwise test involving a context field.
- Optional completion and submission of an existing buffered unit.
- Optional submission of a computed prefix directly from the input address.
- Optional handling of a trailing portion through a separate helper.
- Updates to a buffered-state field and a caller-supplied progress word.

The apparent intent is buffered, block-oriented delivery. That interpretation is **inferred**, not a guarantee that every accepted context describes a valid block size or that the unidentified helper performs a particular memory-copy operation.

This contract describes static evidence only. No runtime tests have been performed.

## Interface

### Inputs and result

The function accepts five arguments:

| Argument | Established representation and use |
|---|---|
| Context address | A 32-bit address of mutable state, accessed using the layout below. |
| Delivery position | A 32-bit value passed to the delivery callback. After a successful submission of an existing buffered unit, it is advanced by that unit’s configured size. |
| Progress address | A 32-bit address at which the function writes a 32-bit progress value on specified paths. |
| Input address | A 32-bit address passed to the callback or helper. Address advancement uses 32-bit arithmetic. |
| Input count | A 32-bit unsigned value that is first interpreted as signed for validation. |

Normal return values are exactly:

- **1:** The selected path completed without a callback returning zero.
- **0:** A delivery callback returned zero.

A nonzero callback result is treated as success; it need not equal 1.

A return value of 1 does not independently establish that all input bytes were delivered. Some paths stage data, and the final staging helper’s transfer extent is not established by the supplied evidence.

### Context and descriptor layout

The observed layout uses 32-bit words and 32-bit addresses:

| Location | Observed role |
|---|---|
| Context + 0x00 | Address of a descriptor. |
| Context + 0x0c | Buffered-state word, initially interpreted as a signed 32-bit quantity. |
| Context + 0x30 | Address used as the internal buffer base. |
| Context + 0x68 | Word used in route selection when the initial buffered-state word is zero. |
| Descriptor + 0x04 | Configured unit size, read as unsigned 32-bit. |
| Descriptor + 0x18 | Delivery callback address. |

These offsets establish access locations, not a complete object definition. In particular, they do not prove allocation sizes, ownership, or that every nominal buffer address has sufficient backing storage.

The route-selection word is tested as raw bits. The function does not dereference the value stored at context + 0x68.

### Arithmetic and validity requirements

All observed integer and address calculations are 32-bit. Signed interpretation is material in the input validation, the early-buffering comparison, and the decision to submit a computed prefix.

The caller must provide valid storage for every access reached on its selected path, including a writable progress word when a progress write occurs. Required source and destination extents additionally depend on the helper’s unknown behavior.

No null-pointer, bounds, alignment, overlap, descriptor-validity, or callback-validity checks are established.

## Behavior

### 1. Validate the input count

Before reading context fields, the function interprets the input count as signed 32-bit and requires it to be greater than zero.

Consequently:

- Counts from 1 through 0x7fffffff pass.
- Zero fails.
- Counts from 0x80000000 through 0xffffffff fail, despite their unsigned representation.

Failure invokes the fatal dependency with two zero-valued arguments and a diagnostic concerning a positive input length. The supplied evidence marks this call as nonreturning. No normal status return or explicit progress write precedes it.

### 2. Select the route

The buffered-state word at context + 0x0c is read once for initial decisions.

- If it is nonzero, the buffered route is selected without consulting context + 0x68.
- If it is zero, the word at context + 0x68 is read. The buffered route is selected when its bitwise AND with the original input count is nonzero.
- Otherwise, the direct route is selected.

This is a literal bitwise test. It is not established as a modulo, alignment, or divisibility test.

### 3. Direct route

The function resolves the callback through the current descriptor address and calls it with:

1. The context address.
2. The original delivery position.
3. The original input address.
4. The original input count.

After the callback returns:

- A zero result causes an explicit progress write of zero, followed by return 0.
- A nonzero result causes an explicit progress write of the original input count, followed by return 1.

This route does not explicitly read the configured unit size or write the buffered-state field. Any changes made by the callback remain outside that statement.

### 4. Enter the buffered route

The configured unit size is read through the descriptor address at context + 0x00.

An unsigned size greater than 32 invokes the fatal dependency with two zero arguments and a diagnostic concerning buffer capacity. This check precedes the buffered route’s explicit progress writes and helper calls.

The function does **not** establish that the unit size is:

- Nonzero.
- A power of two.
- At least as large as the existing buffered state.

Those omissions are behaviorally significant.

If the initial buffered state is zero, the function explicitly writes progress zero and proceeds to the prefix-and-trailing-portion calculation below.

### 5. Append without submitting a unit

When the initial buffered state is nonzero, the function adds that state to the input count using 32-bit arithmetic. It compares the signed interpretation of this sum with the signed interpretation of the configured unit size.

If the sum is strictly smaller, the function invokes the staging helper with five displayed arguments:

1. Internal buffer base advanced by the initial buffered-state value.
2. Current input address.
3. Original input count.
4. Current input address again.
5. Current input address again.

After the helper returns, the function rereads the buffered-state word, adds the original input count, and writes the result back. It then writes progress zero and returns 1.

The reread matters: the stored state is based on the value present **after** the helper call, not necessarily the initially observed value.

Equality does not take this path. A sum equal to the configured unit size proceeds to unit submission.

### 6. Complete and submit an existing buffered unit

For a nonzero initial buffered state that does not take the preceding early return, the function computes a gap as the configured unit size minus the initial buffered state, using 32-bit arithmetic.

It invokes the staging helper with:

1. Internal buffer base advanced by the initial buffered state.
2. Current input address.
3. The computed gap.
4. Current input address again.
5. Current input address again.

After that helper returns, it resolves the delivery callback through the context’s then-current descriptor and submits:

- The context.
- The current delivery position.
- The internal buffer base.
- The previously read configured unit size.

If this callback returns zero, the function immediately returns 0. On this failure path, the function itself has not yet written progress. Any previous progress value can therefore remain, subject to writes by dependencies or aliasing.

On success, the function:

- Subtracts the gap from the remaining input count.
- Advances the input address by the gap.
- Advances the delivery position by the configured unit size.
- Writes the configured unit size to progress.

The progress value therefore includes the full submitted buffered unit, including bytes that may have been present before this invocation. It is not simply a count of bytes consumed from the current input.

### 7. Compute and submit a prefix

The function computes a trailing quantity by taking the remaining input count bitwise AND with the configured unit size minus one. Both subtraction and masking use 32-bit values.

It then subtracts that trailing quantity from the remaining count to obtain a prefix quantity.

This is established as a bitmask calculation. It is equivalent to the usual remainder-and-whole-prefix split only under additional conditions, such as a suitable power-of-two unit size.

If the prefix quantity is positive when interpreted as signed 32-bit, the function resolves the callback again through the current descriptor and submits:

- The context.
- The current delivery position.
- The current input address.
- The prefix quantity.

A zero result returns 0 immediately. There is no explicit buffered-state update after this failure.

A nonzero result causes the function to reread progress, add the prefix quantity using 32-bit arithmetic, and write the sum back. Thus callback changes to progress, including changes through aliases, can affect the accumulated value.

A prefix quantity that is zero or signed-negative is not submitted.

### 8. Handle the trailing portion and store state

If the trailing quantity is nonzero, the function invokes the staging helper again. At this call site, the supplied evidence shows only two arguments:

1. The internal buffer base.
2. The current input address advanced by the computed prefix quantity.

No explicit transfer length is established for this invocation. It is therefore not justified to specify that exactly the trailing quantity is copied.

After this optional helper call returns, the function writes the computed trailing quantity to context + 0x0c and returns 1.

This final store overwrites any intervening dependency change to that field. The stored value is based on the configured unit size read earlier, even if a dependency has since changed the descriptor.

## Side effects

The function’s explicit effects are:

- Reads of the context, descriptor, and progress locations described above.
- Writes to the progress word on path-specific success and failure points.
- Writes to context + 0x0c on the append-only and completed buffered paths.
- Calls to the fatal, staging, and delivery dependencies.

The function itself does not visibly allocate or release memory. No such guarantee can be extended to its dependencies.

Failure is not transactional:

- A staging helper can run before a delivery callback fails.
- A successful buffered-unit submission can precede a failed prefix submission.
- A failure can leave progress unchanged, zeroed, or reflecting an earlier successful submission, depending on the path.
- Dependency mutations are not rolled back.

No non-aliasing constraint is checked. If progress storage overlaps context fields, descriptor storage, or other dependency-visible memory, the explicit writes can affect subsequent behavior. The ordered accesses above remain relevant; treating progress as an independent counter requires an additional caller-side assumption.

The context and input addresses are passed to external code. Their ownership, lifetime after the call, retention, and release behavior are not established.

## Dependencies

The following opaque IDs are assigned locally for otherwise unidentified dependencies; they do not assert externally verified identities.

| ID | Established interaction |
|---|---|
| F-00000001 | Fatal dependency. Called with two zeros and a diagnostic on either validation failure. Marked nonreturning in the supplied evidence. |
| F-00000002 | Staging helper. Two call sites display five arguments; the trailing-portion call site displays only two. Its complete calling convention and memory effects are unknown. |
| F-00000003 | Indirect delivery dependency selected from descriptor + 0x18. Receives context, delivery position, data address, and count. Zero means failure to this function; any nonzero result means success. |

F-00000003 is resolved at each callback site rather than captured once for the entire invocation. Helper or callback mutations to the context’s descriptor can therefore change which callback a later submission uses.

Nothing establishes that any dependency preserves globals, context contents, descriptor contents, progress storage, or input memory.

## Acceptance examples

These are conditional static examples, not observed executions. Unless stated otherwise, they assume valid disjoint storage, returning staging calls, and dependencies that do not mutate the described bookkeeping.

1. **Zero-length rejection.** With input count zero, F-00000001 is invoked before any context read or explicit progress write. There is no normal return.

2. **High-bit count rejection.** Input count 0x80000001 also invokes F-00000001 because its signed interpretation is negative.

3. **Direct success.** With buffered state zero and a route-selection word whose AND with count 7 is zero, F-00000003 receives the original position, input address, and count 7. A nonzero result produces progress 7 and return 1.

4. **Direct failure.** Under the same routing conditions, a zero callback result produces progress zero and return 0.

5. **Append below capacity.** With buffered state 3, configured size 8, and input count 2, the append-only path invokes F-00000002 at internal buffer base + 3. If the helper preserves the state word, the final buffered state is 5, progress is zero, and the return is 1.

6. **Exact completion.** With buffered state 3, size 8, and count 5, the strict comparison does not take the append-only path. The helper is called with gap 5, followed by a callback for 8 bytes from the internal buffer. On success, progress is 8, buffered state becomes zero, and the return is 1.

7. **Failure while submitting existing buffered data.** In the preceding example, a zero callback result returns 0 before any explicit progress write by this function. An initial progress value of 99 remains 99 only if dependencies and aliases leave it unchanged.

8. **Unit plus direct prefix plus tail.** With buffered state 3, size 8, and count 22, the gap is 5. A successful buffered-unit callback contributes progress 8. The remaining count is 17, giving prefix 16 and trailing quantity 1. A successful prefix callback advances progress to 24. The trailing helper is then called with the buffer base and original input address + 21; its transfer extent remains unknown. The final buffered state is 1.

9. **Non-power-of-two size.** With initial state zero, size 6, count 10, and routing bits that select buffering, the trailing quantity is 10 AND 5, which is zero. The computed prefix is 10, not a multiple of 6. The function submits that prefix if the callback is reached successfully.

10. **Zero configured size.** With initial state zero, count 3, and buffering selected, size zero passes the upper-bound check. The trailing quantity becomes 3 and the prefix becomes zero. No delivery callback is made; the two-argument staging call occurs, and the final buffered state is 3 if that helper returns.

## Uncertainties

- **documented_only — Staging semantics:** The helper resembles a memory-transfer operation, but its exact operation, overlap behavior, error behavior, and final-call length are unknown.
- **documented_only — Delivery semantics:** The position may represent an output offset or another routing value. Its units and external effects are not established.
- **documented_only — Ownership:** No dependency’s retention, allocation, or release policy is known.
- **documented_only — Valid state domain:** Positive power-of-two sizes and buffered states smaller than the configured size appear consistent with the inferred buffering intent, but are not enforced as general preconditions.
- **documented_only — Arithmetic hazards:** Invalid or extreme state values can produce wrapped gaps, addresses, and signed comparisons. Their safety cannot be established from this function alone.
- **documented_only — Fatal behavior:** Nonreturning behavior is supplied by the static representation, not independently observed.
- **documented_only — Abnormal dependency exits:** Exceptions, faults, nonlocal transfers, and partial dependency effects are outside the established normal-return behavior.

## Evidence

The supplied static function representation establishes the field offsets, constants, branch conditions, 32-bit operations, explicit stores, callback arguments, and normal return values described here.

The block-oriented purpose and staging interpretation are inferences, separated from those concrete observations. No dependency implementation, full helper prototype, caller contract, allocation metadata, or runtime trace was supplied.

No runtime tests have been performed.
