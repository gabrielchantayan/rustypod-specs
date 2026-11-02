<!-- Trusted provenance: F-eca0b6384dc0; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-eca0b6384dc0

## Purpose

F-eca0b6384dc0 conditionally decreases a 16-bit count. When the requested decrease is positive but smaller than the existing count, it also invokes an unresolved indirect dependency with two address-like arguments and a masked byte quantity.

The apparent intent is to discard a prefix of four-byte elements and reposition the remaining elements. That interpretation is **inferred intent**, not an established contract for the indirect dependency. The available evidence does not establish that the dependency copies memory, supports overlapping ranges, or interprets its arguments as destination, source, and length.

The established behavior is narrower:

- A zero request causes an immediate return without accessing the supplied record.
- A request at least as large as the count clears the count without invoking the dependency.
- A smaller positive request decreases the count, then invokes the dependency.
- The dependency’s third argument represents only the low 14 bits of the remaining count, multiplied by four. It does not necessarily describe all remaining elements.

## Interface

**Inputs**

1. **Requested decrease:** an unsigned integer, analyzed here as 32 bits.
2. **Record address:** an address through which a 16-bit unsigned count is accessed.

The reconstructed layout relevant to this operation is:

| Byte offset | Width | Established use |
|---|---:|---|
| 0 | 16 bits | Unsigned count, read and possibly replaced |
| 2 | 16 bits | No direct access established |
| 4 | 32 bits | Address-like value used only for the indirect invocation |

The offset-four field is accessed through a reconstructed 32-bit integer load. Its role as a buffer address is inferred from its use, not established by a declared record type.

The width model above follows the supplied reconstruction: a 16-bit unsigned count and 32-bit integer arithmetic. Exact binary type definitions, pointer representation, byte order, and alignment requirements are not independently supplied. These limitations matter when interpreting the offset-four value as an address.

**Output**

The function has no result value. Observable outcomes are record modification, possible dependency invocation, and any effects or failures of that invocation.

**Caller preconditions**

- For a zero request, the record need not be readable or writable because this function does not access it.
- For a nonzero request, the count must be readable and writable for defined ordinary execution.
- For the partial-decrease branch, the offset-four field must also be readable.
- Any additional requirements imposed by the indirect dependency remain unknown.

The function does not validate storage accessibility, alignment, allocation extent, or the validity of the indirect target.

## Behavior

### Zero request

If the requested decrease is zero, the function returns immediately.

This branch does not read the count, write the record, load the offset-four field, or invoke the indirect dependency. An invalid or null record address is therefore not itself exercised by this branch.

### Request equal to or greater than the count

For a nonzero request, the function reads the unsigned 16-bit count and compares it with the full unsigned request.

If the request is equal to or greater than the count:

- The count is replaced with zero.
- The function returns.
- The offset-four field is not directly accessed.
- The indirect dependency is not invoked.

The request is not narrowed to 16 bits before comparison. A large request therefore clears the count rather than wrapping into a smaller request. In particular, any request of at least 65,535 takes this branch for every possible 16-bit count.

A nonzero request against an already-zero count still performs the zero store. This differs observably from the zero-request branch, which performs no record access.

### Positive request smaller than the count

When the request is positive and strictly smaller than the count, the function calculates the remaining count by subtraction.

Under the ordinary precondition that the count is not externally changed between its represented reads:

- The original count is between 2 and 65,535.
- The request is between 1 and 65,534.
- The remaining count is between 1 and 65,534.
- The subtraction does not underflow.

The function stores the remaining count into the 16-bit count field **before** invoking the indirect dependency.

The reconstructed arithmetic retains the low 16 bits of the subtraction result. Under the stable-count precondition, the remaining count already fits in 16 bits, so that reduction does not change its value. No saturation operation or error check is present.

After updating the count, the function invokes the unresolved dependency with three arguments:

1. A 32-bit value obtained from the offset-four field.
2. A separately obtained offset-four value plus four times the request.
3. Four times the low 14 bits of the calculated remaining count.

The first and second arguments should not be described unconditionally as equal-base addresses. The evidence represents two accesses to the same field, not a guaranteed single snapshot reused twice.

### Argument arithmetic

Under the 32-bit unsigned arithmetic model, the second argument’s addition has modulo-\(2^{32}\) behavior. No address-overflow check is shown.

On the stable partial-decrease path, the request cannot exceed 65,534, so multiplying the request by four does not itself overflow 32 bits. Adding that displacement to a sufficiently large field value can nevertheless wrap.

The third argument is bounded between 0 and 65,532 inclusive and is always a multiple of four. Its behavior repeats every 16,384 remaining elements:

- Remaining counts from 1 through 16,383 produce their full count multiplied by four.
- A remaining count of 16,384 produces zero.
- A remaining count of 16,385 produces four.
- A remaining count of 32,768 or 49,152 also produces zero.

A zero third argument does **not** suppress the indirect invocation. Whether the dependency treats that argument as a zero-length no-op is unknown.

### Access ordering

The following ordering is established by the supplied representation:

1. The request is checked for zero before record access.
2. The count is read for the nonzero-request comparison.
3. In the partial-decrease branch, a further count read is represented for subtraction.
4. The replacement count is stored before dependency argument preparation and invocation.
5. The dependency is invoked once on that branch.
6. If the invocation returns normally, this function returns without another explicit record update.

The relative order of the two offset-four reads is not established. Neither is their order relative to reading the indirect target.

These are source-facing observations. The evidence does not establish exact machine-load counts, volatile access semantics, atomicity, or synchronization. Concurrent modification cannot safely be treated as supported behavior.

## Side effects

**Direct record modification**

For every nonzero request that reaches a valid count store, the function writes the count field:

- Zero for an equal-or-larger request.
- The calculated remainder for a smaller request.

No direct writes to the offset-two field, offset-four field, or pointed-to storage are established.

**Indirect effects**

The partial-decrease branch invokes an unresolved dependency. Its effects on memory, global state, resources, control flow, and the record itself are unknown.

The updated count is already visible when that dependency begins. There is no explicit rollback. If the dependency fails, does not return, or modifies the record again, the available evidence provides no stronger final-state guarantee.

**Ownership**

No direct allocation, deallocation, ownership transfer, reference-count update, or resource release is established. The function does not directly clear or replace the offset-four value.

The indirect dependency cannot be assumed to preserve ownership or leave globals unchanged. Conversely, it cannot be asserted to release or take ownership of anything.

**Errors**

There is no explicit error return, validation diagnostic, or recovery path. Invalid memory accesses or an invalid indirect target may prevent normal completion. Their precise manifestation is outside the supplied evidence.

## Dependencies

### F-unresolved-01

This is a contract-local opaque identifier for the unresolved indirect dependency; it is not a recovered function identity.

Established invocation conditions and arguments:

- Invoked exactly once when the request is positive and smaller than the stable count.
- Invoked only after the new count has been stored.
- Receives three arguments: an offset-four value, a separately loaded offset-four value advanced by four times the request, and the masked byte quantity.
- Any returned value is unused.

Unknown properties include argument meaning, calling convention, overlapping-range behavior, memory accessibility requirements, ownership effects, global-state effects, and failure behavior.

No named library operation can be substituted for this dependency on the available evidence.

## Acceptance examples

These examples are conditional predictions from static evidence, not executed tests. Unless stated otherwise, assume valid accessible storage, stable fields, the stated width model, and a dependency that returns normally.

| Initial count | Request | Direct count store | Dependency invocation |
|---:|---:|---:|---|
| Any value | 0 | None | None |
| 0 | 1 | 0 | None |
| 7 | 7 | 0 | None |
| 7 | 8 | 0 | None |
| 10 | 3 | 7 | Base value; base plus 12; third argument 28 |
| 16,385 | 1 | 16,384 | Base value; base plus 4; third argument 0 |
| 16,386 | 1 | 16,385 | Base value; base plus 4; third argument 4 |
| 65,535 | 1 | 65,534 | Base value; base plus 4; third argument 65,528 |
| 65,535 | 65,536 | 0 | None |

Additional conditional acceptance cases:

- **Zero request with an invalid record address:** no record access or dependency invocation is attributable to this function.
- **Nonzero request with a zero count:** a zero store is performed; no offset-four read is required.
- **Partial decrease with a wrapping base:** if the stable offset-four value is 4,294,967,292 and the request is one, the second argument is zero under 32-bit arithmetic. No protective rejection is established.
- **Dependency observes the record:** it encounters the count after the direct reduction, subject to any external interference.
- **Dependency changes the count:** this function performs no explicit restoration afterward; the direct store is not an unconditional guarantee of the final count.

## Uncertainties

**documented_only — Dependency semantics.** The invocation resembles movement of four-byte elements, but copying, overlap safety, destination/source ordering, and byte-count interpretation are unconfirmed.

**documented_only — Record definition.** Only the accessed offsets and widths are available. Total record size, padding, packing, and the meaning of the offset-two field are unknown.

**documented_only — Binary representation.** The contract uses the reconstructed 16-bit and 32-bit model. Exact target ABI, byte order, pointer conversion behavior, and alignment rules are not independently established.

**documented_only — Concurrent changes.** Multiple represented reads mean a changing count or offset-four field could invalidate the stable-input relationships above. No synchronization or supported concurrent-update protocol is shown.

**documented_only — Indirect control flow.** The target is unresolved. Successful return, exception behavior, and effects beyond the argument interface cannot be guaranteed.

**Inferred intent.** Reducing a count and advancing an address by four times the request suggests removal of an element prefix. The 14-bit mask prevents treating that interpretation as an unrestricted, full-remainder movement contract.

## Evidence

The supplied static representation establishes the zero-request early return, unsigned count comparison, two count-update branches, pre-invocation count store, offset-four accesses, four-byte displacement, 14-bit mask, and unresolved three-argument invocation.

No external dependency implementation, binary inspection, ABI definition, or runtime observation was supplied. No runtime tests have been performed. Acceptance examples are arithmetic and control-flow consequences of the stated model, not observed execution results.
