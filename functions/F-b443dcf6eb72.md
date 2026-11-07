<!-- Trusted provenance: F-b443dcf6eb72; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-b443dcf6eb72

## Purpose

F-b443dcf6eb72 classifies a signed coordinate pair against bounds stored in a mutable record, optionally adjusts the first coordinate, conditionally invokes F-5f57ab3ca437, and commits the resulting classification and coordinates.

The established classification is asymmetric:

- The second coordinate must lie within a lower-inclusive, upper-exclusive interval.
- The first coordinate must be strictly below an upper bound.
- A first coordinate below its lower bound is not rejected. Instead, it is replaced with one less than that lower bound.

The function also clears two record fields under conditions determined before the dependency call.

Interpreting this operation as coordinate tracking, hit testing, or change notification is inferred intent, not an established interface guarantee. The record fields and dependency have no supplied semantic definitions.

## Interface

### Inputs and result

The function accepts three inputs:

1. A record address represented by a 32-bit input in the supplied representation.
2. A signed 32-bit first coordinate, called **x** in this contract.
3. A signed 32-bit second coordinate, called **y** in this contract.

It returns no value. Its observable results are record accesses, record updates, and any effects of F-5f57ab3ca437.

For ordinary completion, the record must remain valid for every access actually performed, including accesses after the dependency returns. No null check, record-size check, or pointer validation is visible.

### Record layout

Offsets below are byte offsets from the record address. Every listed field occupies four bytes.

| Offset | Contract name | Use |
|---|---|---|
| `0x0c` | x lower bound | Signed comparison; source for the below-bound replacement |
| `0x10` | x upper bound | Signed, exclusive upper limit |
| `0x14` | y lower bound | Signed, inclusive lower limit |
| `0x18` | y upper bound | Signed, exclusive upper limit |
| `0x1c` | auxiliary field A | Written with four zero bytes when clearing is selected |
| `0x20` | auxiliary field B | Written with four zero bytes when clearing is selected |
| `0x24` | stored classification | Read as a 32-bit integer; ultimately written as zero or one |
| `0x28` | stored x | Compared with the effective x; ultimately overwritten |
| `0x2c` | stored y | Conditionally compared with y; ultimately overwritten |

Ordinary contiguous storage therefore needs to extend through byte offset `0x2f`. This does not establish a complete record size or define any unlisted fields.

The supplied information does not establish byte order, alignment requirements, or a portable pointer ABI. The signed comparisons concern 32-bit values. The auxiliary fields need not have a signed interpretation because the function only writes zero to them.

### Preconditions and ownership

The function does not visibly allocate, free, retain, or transfer ownership of the record. It requires writable access to the final state fields and, when clearing occurs, the auxiliary fields.

No ownership or lifetime guarantee can be assigned to F-5f57ab3ca437 from this body. In particular, callers must not assume that the dependency preserves record storage or unrelated global state. If it invalidates the record before returning, the subsequent writes lack a supported valid-memory outcome.

Concurrent mutation is not synchronized by this function. The simplified value-based rules below assume the relevant fields do not change between their reads. The access-order rules describe where separate observations can occur.

## Behavior

### Classification

The newly computed classification is:

- **Zero** when all three signed comparisons hold: y is at least the y lower bound, y is strictly below the y upper bound, and x is strictly below the x upper bound.
- **One** otherwise.

The subtraction-sign and overflow information in the supplied body implements signed ordering. Classification must not be described as depending only on the sign of a wrapped subtraction.

Bounds are examined conditionally, in this order:

1. Read the y lower bound.
2. If that bound exceeds y, classification is one. The y upper and x upper bounds are not read.
3. Otherwise, read the y upper bound.
4. If y is not strictly below that bound, classification is one. The x upper bound is not read.
5. Otherwise, read the x upper bound and classify according to whether x is strictly below it.

There is no normalization or validation of bound ordering. With stable fields, an empty or reversed y interval admits no coordinate pair. An inconsistent x interval is handled by the comparisons and replacement rule as written; no separate error results.

### Effective coordinates

When classification is one, effective x remains the supplied x. Neither the x lower bound nor the stored coordinate fields is read during this part of the operation.

When classification is zero:

1. Read the x lower bound.
2. If supplied x is at least that bound, retain supplied x.
3. Otherwise, read the x lower bound again and derive effective x by subtracting one from this second observation.

Thus the value used for the comparison and the value used for the replacement are separate reads. They agree only if the field remains stable.

The replacement is not a clamp to the lower bound. For an ordinary representable subtraction, every admitted x below a stable lower bound becomes the same sentinel-like value: lower bound minus one.

There is no second classification after replacement. An effective x that fails the original upper-bound test does not change the already computed classification.

The minimum signed 32-bit lower bound presents a subtraction-underflow question. Its boundary semantics are documented under Uncertainties rather than assigned an unsupported portable result.

Effective y is always supplied y.

### Coordinate comparison and clearing decision

A clearing decision initially selects clearing.

For classification one, it remains selected. Stored coordinates are not examined.

For classification zero:

1. Read stored x and compare it with effective x.
2. If they differ:
   - Mark the coordinates as changed.
   - Leave clearing selected.
   - Do not read stored y.
3. If they match:
   - Read stored y and compare it with supplied y.
   - If y differs, mark the coordinates as changed and leave clearing selected.
   - If y also matches, mark the coordinates as unchanged, read stored classification, and select clearing exactly when that observed value is nonzero.

Consequently, an unchanged accepted pair preserves the auxiliary fields only when the classification read for this clearing decision is zero. Any nonzero value selects clearing; the prior field is not assumed to contain only zero or one.

### Dependency-call decision

After the preceding decisions, the function reads stored classification.

It invokes F-5f57ab3ca437 exactly once if either:

- This newly observed stored classification differs from the new classification; or
- Classification is zero and the effective coordinate pair was found to differ from the stored pair.

Otherwise, it does not invoke the dependency.

Important consequences:

- A rejected pair does not trigger a call solely because its coordinates differ.
- An accepted pair with changed effective coordinates triggers a call even if stored classification is already zero.
- An unchanged accepted pair can cause two reads of stored classification: one for clearing, then another for the call decision.
- The dependency is called before this function performs any of its own record writes.

The supplied call shows no explicit arguments and does not use a return value. No undocumented argument or callback protocol is established.

### Commit order

If the dependency returns normally, or no call was needed, the function performs these writes in order:

1. If clearing was selected, write zero to auxiliary field A.
2. If clearing was selected, write zero to auxiliary field B.
3. Write the new classification to `0x24`.
4. Write effective x to `0x28`.
5. Write supplied y to `0x2c`.

The last three writes occur even when they reproduce the existing values.

The clearing decision and effective coordinates are not recomputed after the dependency returns. Dependency changes to fields subsequently written by this function are therefore overwritten by those writes. Dependency changes to auxiliary fields survive only when this function does not clear them.

## Side effects

The direct mutation set is limited to auxiliary fields A and B, stored classification, stored x, and stored y. Bound fields are not directly written.

The visible body provides no explicit error return, exception handling, rollback, allocation, deallocation, logging, or synchronization.

Failure of an address access, abnormal dependency termination, or nonreturn from the dependency has no defined recovery path here. In particular, successful completion of all writes must not be promised if the dependency does not return normally.

The update is not presented as atomic. If clearing occurs, the auxiliary fields are cleared before the new classification and coordinates are committed. Other observers may therefore encounter an intermediate state unless external synchronization prevents observation.

## Dependencies

### F-5f57ab3ca437

**Established static evidence**

- There is one visible call site.
- It can be reached at most once per invocation.
- It executes only under the classification-or-coordinate-change rule above.
- It executes before all direct record writes by F-b443dcf6eb72.
- No explicit arguments are shown.
- No returned value is consumed.

**Inferred intent**

The call may announce or handle a state change. Its position and predicate support this interpretation, but do not establish it.

**Unknown, documented_only**

Its actual purpose, indirect inputs, global effects, memory effects, ownership behavior, reentrancy, failure behavior, and preservation guarantees are unspecified. No preservation of globals or record contents may be assumed.

## Acceptance examples

These are conditional acceptance cases derived from static evidence, not executed tests. Unless stated otherwise, fields remain stable during reads, arithmetic is representable, storage remains valid, and any dependency call returns normally without modifying the record.

Use bounds x lower = 10, x upper = 20, y lower = 30, and y upper = 40.

### Accepted, unchanged pair

Input is x = 15, y = 35. Stored classification is zero and stored coordinates are `(15, 35)`.

- New classification is zero.
- No dependency call occurs.
- Neither auxiliary field is written.
- Classification and both coordinates are nevertheless written again.

### Accepted pair changes

Input is `(16, 35)`. Stored classification is zero and stored coordinates are `(15, 35)`.

- Stored x differs, so stored y is not read.
- The dependency is called once.
- Both auxiliary fields are cleared.
- Final state is classification zero and coordinates `(16, 35)`.

### Below-lower-bound x

Input is `(4, 35)`.

- The pair passes classification because 4 is below the x upper bound and y is in range.
- Effective x becomes 9, not 10.
- Coordinate-change detection compares stored x with 9, not with 4.
- Final stored coordinates are `(9, 35)`.

If stored classification is already zero and stored coordinates are `(9, 35)`, there is no dependency call or auxiliary clearing. Different raw x inputs below the same stable lower bound can therefore produce the same effective state.

### Exclusive upper boundaries

Input `(20, 35)` is rejected because x equals its upper bound. Input `(15, 40)` is rejected because y equals its upper bound.

For either input, if stored classification is already one:

- No dependency call occurs, regardless of stored coordinates.
- Both auxiliary fields are cleared.
- The supplied coordinates are written without x adjustment.

### Inclusive y lower boundary

Input `(15, 30)` receives classification zero. Equality with the y lower bound is allowed.

Input `(15, 29)` receives classification one. In that case, neither the y upper bound nor the x upper bound is read.

### Noncanonical previous classification

Input `(15, 35)` matches stored coordinates, but stored classification is 7.

- Clearing is selected because the prior value is nonzero.
- The dependency is called because 7 differs from the new zero.
- Final classification is normalized to zero.

### No post-replacement reclassification

Let x lower be 10 and x upper be 5, with y still in range. Input x = 4 passes the x upper-bound test, then becomes effective x = 9.

The final classification remains zero despite effective x being above the x upper bound. This follows from operation order, not from an assumption that bounds are well formed.

## Uncertainties

- **Domain meaning — documented_only.** The actual coordinate system, auxiliary-field meanings, and intended consumer of classification are unknown.
- **Subtraction boundary — documented_only.** When the replacement source is the minimum signed 32-bit value, mathematical subtraction by one is outside the signed range. A wrapping 32-bit machine operation would produce the maximum signed value, but a portable source-language overflow guarantee is not established by the supplied representation.
- **Dependency effects — documented_only.** No preservation, lifetime, or ownership assumptions are available.
- **Execution environment — documented_only.** Endianness, alignment, address validity rules, and fault behavior are unspecified.
- **Concurrent or reentrant mutation — documented_only.** Multiple reads can observe different values. There is no visible snapshot or locking guarantee. The ordered accesses remain relevant, but simplified stable-record examples do not cover such interference.
- **Runtime behavior — unverified.** No runtime tests have been performed.

## Evidence

This contract is based solely on the supplied static body for F-b443dcf6eb72 and the supplied opaque dependency mapping.

Established evidence includes four-byte record accesses, signed bound comparisons, conditional reads, the below-lower-bound replacement, separate coordinate and classification decisions, the conditional dependency call, and the ordered final writes.

Domain interpretations are explicitly identified as inferred intent. Missing dependency semantics, environmental guarantees, and overflow-boundary guarantees remain documented_only. No external implementation, runtime trace, test result, or prior behavioral document was used.
