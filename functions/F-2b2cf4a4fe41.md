<!-- Trusted provenance: F-2b2cf4a4fe41; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-2b2cf4a4fe41

## Purpose

F-2b2cf4a4fe41 writes an entry marker into a supplied context, selects a processing path using a 16-bit context field, and submits a four-byte temporary to dependencies.

Three paths are established:

- **Selector 3:** perform byte and nibble replacements based on the index modulo four, interleaved with checked dependency calls.
- **Selector 8:** pass the full 32-bit value to a preparation dependency, then call a distinct storage-facing dependency.
- **Other selectors:** pass the low 16 bits of the value to another preparation dependency, then call the common storage-facing dependency.

The result is a normalized zero/nonzero outcome. Success does not establish persistence, atomicity, or any particular final context contents. Failure does not establish that no changes occurred.

The selector-3 transformations are consistent with updating packed 12-bit entries. That interpretation is **inferred intent**. The established contract is the arithmetic, ordered calls, temporary transformations, and result handling described below.

## Interface

### Inputs

| Input | Established representation | Use |
|---|---|---|
| Context | Address carried by a 32-bit argument | Base for direct field accesses; supplied to storage-facing dependencies. |
| Index | Unsigned 32-bit integer | Used directly outside selector 3; transformed on selector 3. |
| Value | Unsigned 32-bit integer | Supplies replacement bits or a preparation argument. |
| Seed | Four-byte value | Initializes an addressable four-byte temporary. |

The context has two directly accessed fields:

| Context-relative offset | Width | Direct access |
|---|---|---|
| `0x88` | 32 bits | Written with numeric 1 before dispatch. |
| `0x6c` | 16 bits | Read for comparison with 3, then read again for comparison with 8 if the first comparison fails. |

The field meanings are unknown. “Entry marker” describes the visible write, not an established dirty-state or validity protocol.

### Output

The function returns a four-byte value:

- **0** when a checked dependency call returns zero.
- **1** when the selected path completes and its final checked dependency returns nonzero.

Dependency results are tested only for equality with zero. Negative and positive nonzero results are not distinguished by this function.

This contract describes normal returns. Faults, exceptions, non-returning dependencies, and invalid memory accesses have no established recovery behavior.

### Temporary layout

The four-byte temporary is initially the seed. For describing transformations:

- **B0** denotes bits 0–7.
- **B1** denotes bits 8–15.
- **B2** denotes bits 16–23.
- **B3** denotes bits 24–31.

A tuple uses the order **(B0, B1, B2, B3)**. These are numeric byte lanes, not an independently verified memory-byte-order specification.

“Preserve” below means that the function’s stated transformation retains those bits from the temporary immediately before that transformation. It does not mean dependencies preserve them across calls.

### Preconditions and ownership

Defined direct accesses require a context permitting a four-byte write at offset `0x88` and a two-byte read at offset `0x6c`, including any target alignment requirements.

No explicit null check, index bound check, value-range check, or context validation is visible. Dependencies may impose additional preconditions; those are unknown.

The temporary is local to the invocation, and its address is passed to dependencies. The function contains no visible allocation, deallocation, or explicit ownership-transfer operation. Dependency pointer retention and memory-management behavior are unknown. The evidence does not establish that the temporary’s address remains usable after this invocation returns.

## Behavior

### Entry and dispatch

The function performs these direct operations in order:

1. Write numeric 1 to the context’s 32-bit field at offset `0x88`.
2. Initialize the temporary from the seed.
3. Read the 16-bit field at offset `0x6c` and compare it with 3.
4. If unequal, read that field again and compare it with 8.

There is no dependency call between the two selector reads. The representation nevertheless contains two reads, not a single retained selector value. A first read unequal to 3 followed by a second read equal to 8 selects the selector-8 path. If both comparisons fail, the default path is selected.

The function performs no further direct write to the marker field. In particular, it neither restores that field after failure nor reasserts 1 after a dependency returns. **Its final value is unknown once a dependency has executed**, because dependency effects are not constrained by the supplied evidence.

### Selector 3: arithmetic and initial call

The remainder is the original unsigned index masked with 3, producing exactly 0, 1, 2, or 3.

The primary dependency index is calculated by:

- Multiplying the original index by 3 using unsigned 32-bit arithmetic.
- Discarding product bits above bit 31.
- Logically shifting that wrapped product right by two bits.

Wrapping occurs before the shift. The result is therefore not generally equal to the unbounded mathematical quantity ⌊3 × index / 4⌋.

The function calls **F-6801f4e9392e** with, in order:

1. Context.
2. Primary dependency index.
3. Temporary address.
4. Operation argument 0.

A zero result causes an immediate return of 0, without a subsequent temporary transformation or operation-1 call.

A nonzero result permits the remainder-specific processing below. The dependency may have changed the temporary, context, globals, or other reachable state. No particular operation-0 output is established.

For a nonzero remainder, the function computes a secondary index: the primary dependency index plus one, restricted to its low 16 bits. The original primary index is not subject to that 16-bit restriction.

### Selector 3, remainder 0

After the initial call succeeds, the function transforms the current temporary as follows:

- B0 becomes value bits 0–7.
- B1’s low nibble becomes value bits 8–11.
- B1’s high nibble remains unchanged.
- B2 and B3 remain unchanged.

It then calls **F-6801f4e9392e** with the context, primary index, temporary address, and operation argument 1.

A zero result produces 0; a nonzero result produces 1. Only the low 12 bits of the value influence this direct transformation.

### Selector 3, remainder 1

After the initial call succeeds:

- B1’s high nibble becomes value bits 0–3.
- B1’s low nibble remains unchanged.
- B0, B2, and B3 remain unchanged.

The function calls **F-6801f4e9392e** at the primary index with the temporary address and operation argument 1. A zero result immediately returns 0.

On nonzero result, it calls the same dependency at the secondary index, with the same temporary address and operation argument 0. There is no seed reinitialization between calls. Any mutations made by the preceding dependency remain part of the state presented to this call.

If the secondary operation-0 call returns zero, the function returns 0. Otherwise:

- B0 becomes value bits 4–11.
- B1, B2, and B3 remain as they stand after that call.

Finally, it calls **F-6801f4e9392e** at the secondary index with operation argument 1 and normalizes the result.

This path can issue two operation-1 calls. No rollback is visible if the second stage fails.

### Selector 3, remainder 2

After the initial call succeeds:

- B1 becomes value bits 0–7.
- B0, B2, and B3 remain unchanged.

The function calls **F-6801f4e9392e** at the primary index with operation argument 1. A zero result immediately returns 0.

Otherwise, it calls the same dependency at the secondary index with operation argument 0, using the same temporary without reinitialization. A zero result immediately returns 0.

After that call succeeds:

- B0’s low nibble becomes value bits 8–11.
- B0’s high nibble remains unchanged.
- B1, B2, and B3 remain unchanged.

The nibble extraction is governed by unsigned 32-bit shifts: left by 20 with overflow discarded, then logically right by 28. This selects original value bits 8–11, not its uppermost nibble.

A final call to **F-6801f4e9392e** uses the secondary index and operation argument 1. Its result is normalized. There is no visible rollback of earlier effects.

### Selector 3, remainder 3

After the initial call succeeds:

- B0’s high nibble becomes value bits 0–3.
- B0’s low nibble remains unchanged.
- B1 becomes value bits 4–11.
- B2 and B3 remain unchanged.

The final call to **F-6801f4e9392e** uses the **primary index**, the temporary address, and operation argument 1. Its result is normalized.

Although the secondary index is computed on this path, it is not used for a dependency call. There is no second operation-0 call.

The four remainder cases exhaust the possible masked-index results. No additional remainder behavior is reachable under the represented arithmetic.

### Selector 8

When the first selector comparison fails and the second equals 8:

1. Call **F-fe873c951594** with the temporary address and the full unsigned 32-bit value.
2. Call **F-690187c5daac** with the context, original index, and the same temporary address.
3. Normalize the second call’s result.

No result from the preparation call is checked. There is no conditional abort based on its return value. Its temporary layout, memory effects, and possible non-returning behavior are unknown.

### Default path

When neither selector comparison matches:

1. Call **F-e02b89627b27** with the temporary address and the value restricted to its low 16 bits.
2. Call **F-6801f4e9392e** with the context, original index, temporary address, and operation argument 1.
3. Normalize the latter call’s result.

The preparation argument has only 16 significant bits; the evidence does not require describing its calling-convention slot as a two-byte slot.

No preparation result is checked, and there is no preliminary operation-0 call.

## Side effects

The established direct effects are:

- The initial 32-bit context-field write.
- Initialization and selected transformations of the local temporary.
- Ordered dependency calls carrying the specified arguments.

The function does not directly restore the entry marker, roll back an earlier dependency call, allocate memory, or release memory.

Dependency side effects remain unrestricted by this evidence. In particular, dependencies are not assumed to:

- Preserve the marker, selector, other context fields, or globals.
- Preserve temporary bytes not explicitly needed for an operation.
- Leave state unchanged when returning zero.
- Complete operations atomically.
- Avoid retaining pointers.
- Avoid allocating or releasing memory.

Operation arguments 0 and 1 are consistent with read and write roles, respectively. Those roles are inferred, not established dependency contracts.

A return of 0 establishes a checked zero result, not absence of effects. Even the first failed selector-3 call occurs after the direct marker write. Conversely, the initial write does not establish that the marker still equals 1 after that call.

## Dependencies

| Opaque function ID | Established use |
|---|---|
| **F-6801f4e9392e** | Receives context, index, temporary address, and operation argument 0 or 1. Its result is checked for zero. Used throughout selector 3 and for the default final operation. |
| **F-fe873c951594** | Receives temporary address and full 32-bit value on selector 8. No result is checked. |
| **F-e02b89627b27** | Receives temporary address and the value masked to 16 significant bits on the default path. No result is checked. |
| **F-690187c5daac** | Receives context, original index, and temporary address on selector 8. Its result is checked for zero. |

Their implementations were not supplied. Storage units, ownership rules, result taxonomy, and guarantees about successful or failed operations are **documented_only**.

## Acceptance examples

These are conditional static examples, not executed tests. Unless stated otherwise, the selector reads select the indicated path and all calls required to reach an illustrated step return nonzero.

### Remainder 0

With selector 3, index 0, and value `0xABC`, the primary index is 0.

If the initial operation-0 call leaves `(0x12, 0xD3, 0x56, 0x78)`, the operation-1 call at index 0 receives `(0xBC, 0xDA, 0x56, 0x78)`.

A nonzero final result yields 1.

### Remainder 1

With selector 3, index 1, and value `0xABC`, the primary index is 0 and the secondary index is 1.

If the initial call leaves `(0x12, 0xD3, 0x56, 0x78)`, the first operation-1 call receives `(0x12, 0xC3, 0x56, 0x78)`.

If the subsequent operation-0 call at index 1 leaves `(0x45, 0x67, 0x89, 0xEF)`, the final operation-1 call receives `(0xAB, 0x67, 0x89, 0xEF)`.

### Remainder 2

With selector 3, index 2, and value `0xABC`, the primary index is 1 and the secondary index is 2.

An initial operation-0 output of `(0x12, 0xD3, 0x56, 0x78)` leads to a first operation-1 input of `(0x12, 0xBC, 0x56, 0x78)`.

If the next operation-0 call leaves `(0xD5, 0x67, 0x89, 0xEF)`, the final operation-1 input is `(0xDA, 0x67, 0x89, 0xEF)`.

### Remainder 3

With selector 3, index 3, and value `0xABC`, the primary index is 2.

An initial operation-0 output of `(0x12, 0xD3, 0x56, 0x78)` leads to an operation-1 input of `(0xC2, 0xAB, 0x56, 0x78)` at index 2. No call uses the computed secondary index.

### Arithmetic boundaries

For selector 3 and index `0xFFFFFFFF`:

- The wrapped product is `0xFFFFFFFD`.
- The primary index is `0x3FFFFFFF`.
- The remainder is 3.
- Both dependency calls use `0x3FFFFFFF`.

For selector 3 and index `0x00015555`:

- The primary index is `0x0000FFFF`.
- The remainder is 1.
- The secondary index is 0 after 16-bit truncation.
- If preceding calls succeed, the operation-1 indices are `0x0000FFFF` and 0, in that order.

### Failure, mutation, and dispatch

- If the first selector-3 operation-0 call returns zero, the function returns 0 without an operation-1 call. It wrote 1 to offset `0x88` before the dependency and performs no subsequent direct restoration. The field’s value after the dependency is unknown.
- If that failing dependency changes the marker to 7 before returning zero, this function does not overwrite that change. Under the additional condition that nothing else changes the field, it remains 7 on return.
- On remainder 1, if the first operation-1 call succeeds but the next operation-0 call returns zero, the function returns 0 without a final operation-1 call or visible rollback.
- On selector 8, value `0x12345678` reaches **F-fe873c951594** unchanged.
- On the default path, the same value reaches **F-e02b89627b27** as `0x5678`.
- A final checked result of any nonzero magnitude yields 1, not that magnitude.

## Uncertainties

The following remain **documented_only**:

- Context type, semantic field names, valid object extent, and alignment guarantees.
- Dependency index units, valid ranges, storage interpretation, and synchronization rules.
- Target memory byte order and portability beyond the represented widths.
- Whether operation arguments 0 and 1 actually denote reading and writing.
- Dependency effects on context, globals, temporary contents, and other memory.
- Final marker contents after any dependency, absent additional conditions.
- Preparation output layouts and how many temporary bytes they replace.
- Whether seed-derived bytes survive in actual executions.
- Dependency pointer retention, allocation, release, and ownership behavior.
- Fault, exception, signal, and non-returning behavior.
- Concurrent selector changes and other shared-state interactions.

The inferred packed-entry purpose does not justify replacing the wrapped arithmetic, removing secondary-index truncation, assuming dependency preservation, or treating the multi-call paths as transactions.

## Evidence

This contract is derived solely from the supplied static function representation and opaque dependency mapping.

Established static evidence covers direct field accesses, represented widths, selector comparisons, unsigned arithmetic, byte and nibble replacements, call ordering and arguments, zero-result checks, and normalized returns. In particular, the marker is written before dependencies, and no later direct marker write is present; its final value is not established.

The storage interpretation is inferred intent. Dependency semantics and target-layout details remain unresolved.

No runtime tests, execution traces, dependency implementations, or target-memory observations were supplied or performed. The acceptance examples are derived conditional expectations, not claims of observed execution.
