<!-- Trusted provenance: F-40cbf7b4e27b; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-40cbf7b4e27b

## Purpose

F-40cbf7b4e27b conditionally prepares temporary storage, obtains an unsigned 32-bit result from it, and compares that result with an unsigned 16-bit field in a caller-supplied context.

On normal return, the result is:

- **1** if execution reaches the comparison and the two values differ.
- **0** if any preceding gate fails or the comparison finds equality.

A result of 0 therefore does not establish equality. The comparison may not have occurred.

The supplied static representation establishes the access widths, gates, and dependency-call sequence described below. Business meanings such as validation, change detection, and resource matching are not established.

## Interface

### Arguments

| Position | Contract name | Representation and use |
|---|---|---|
| 1 | Context | Address-bearing value used as the base of fixed-offset memory accesses. |
| 2 | Signed quantity | Signed 32-bit value. Zero prevents processing. Negative values are negated before a signed threshold test, with an unresolved minimum-value case. |
| 3 | Payload | Opaque 32-bit value. Written into the first temporary word and separately passed by value to F-9d9e918de01f. |
| 4 | Companion | Signed 32-bit value. The value −1 prevents processing; otherwise its bit pattern is written into the second temporary word. |

These widths follow the supplied representation. An independently verified machine-level calling convention is unavailable.

### Context accesses

| Offset from context | Access width | Use |
|---|---:|---|
| `0xE8` | 32 bits | First argument to each reached call of F-5710a8628cba. |
| `0x160` | Unsigned 16 bits | Comparison operand read after F-9e5fecb2740f returns. |
| `0x168` | 8 bits | Entry flag; zero disables processing. |
| `0x1A0` | 32 bits | Third argument to each reached call of F-5710a8628cba. |
| `0x1D8` | 32 bits | First argument to F-9d9e918de01f. |

These are access requirements, not a complete object layout. No total object size or semantic relationship between fields is established.

### Global selector inputs

Two distinct global storage locations supply the second arguments to the lookup dependency:

- **Selector storage A** is read when the first lookup is reached.
- **Selector storage B** is read when the second lookup is reached.

“Selector A” and “selector B” below mean the values obtained from those respective reads. The storage locations are distinct; their stored values need not differ.

Neither selector is established as an immutable constant. Dependencies may change either location. The supplied material does not independently establish the selector-storage declarations or exact read widths.

### Temporary storage

The supplied representation shows two adjacent 32-bit temporary words, forming an eight-byte initialized region:

| Relative offset | Initial contents |
|---|---|
| `0` | Payload bit pattern |
| `4` | Companion bit pattern |

The address of the first word is passed to processing dependencies. The initialized region does not establish how much storage those dependencies may access or what object layout they expect.

### Return value and preconditions

The function returns a 32-bit value restricted to 0 or 1 on normal return. It exposes no separate error code.

The context must permit the initial byte read at `0x168`, even if the quantity is zero or the companion is −1. Other context fields and global selector storage must be readable when their corresponding paths are reached.

A non-null first lookup result must support a 32-bit read at its starting address. Dependency-specific validity requirements remain unknown.

There is no explicit null-context check, access-fault recovery, or general dependency-failure recovery.

## Behavior

### Entry gates

The function first reads the byte at context offset `0x168`. Further processing requires, in short-circuit order:

1. A nonzero flag byte.
2. A companion other than −1.
3. A nonzero signed quantity.

Failure of any condition returns 0 without calling a dependency, initializing the temporary words, or reading either global selector location.

The flag’s signed interpretation is irrelevant: only zero versus nonzero is tested.

### Temporary initialization and first lookup

After all entry gates pass, the function writes the payload into the first temporary word, then writes the companion into the second.

It then calls F-5710a8628cba using:

1. A 32-bit read from context offset `0xE8`.
2. A read from selector storage A.
3. A 32-bit read from context offset `0x1A0`.

These reads supply this first call’s arguments. Their relative evaluation order is not established: in particular, the global selector read is not known to precede, follow, or occur between the two context reads.

If the lookup result is null, the function returns 0 without dereferencing it. Otherwise, the function reads the first 32-bit word at the returned address. A zero word also causes return 0.

The first lookup result has no later visible use. The function does not explicitly release it.

### Quantity normalization and threshold

Only after the first lookup returns a non-null address containing a nonzero first word does the function examine the quantity’s sign.

A negative quantity is replaced by its negation; a nonnegative quantity remains unchanged. For every signed 32-bit input except the minimum value, this produces its representable absolute magnitude.

The resulting signed value must be at least 16. A value below 16 returns 0 without calling F-9d9e918de01f. Consequently, quantities from −15 through −1 and from 1 through 15 can still cause the first lookup, but cannot reach the dependent predicate.

The input −2,147,483,648 is unresolved. Its positive magnitude is not representable in a signed 32-bit value. The supplied representation shows signed negation and a subsequent signed comparison, but does not establish a portable overflow outcome. No unconditional return value or later call sequence is specified for this input.

### Dependent predicate

If the normalized quantity passes the threshold, the function reads the 32-bit field at context offset `0x1D8` and calls F-9d9e918de01f with:

1. That field value.
2. The original payload argument.

The payload is passed directly by value, not read back from temporary storage.

A zero result returns 0. Any nonzero result permits the second lookup. No meaning is established for individual nonzero results.

The `0x1D8` read occurs after the first lookup returns. The first lookup is not assumed to preserve that field.

### Second lookup and temporary processing

After a nonzero predicate result, the function calls F-5710a8628cba again using:

1. A fresh 32-bit read from context offset `0xE8`.
2. A read from selector storage B.
3. A fresh 32-bit read from context offset `0x1A0`.

The relative evaluation order of these three reads is not established.

This call uses selector storage B, not a cached value from selector storage A. The two context fields are also read again rather than reused from the first call. Intervening dependencies may have changed any of these locations; no global-preservation guarantee is assumed.

The second lookup result is forwarded without a visible null check or pointed-word check. Processing then proceeds in this order:

1. F-7afe554bcf62 receives the temporary address and the second lookup result.
2. F-abb7a1ff7712 receives the temporary address and a 32-bit zero argument.
3. F-9e5fecb2740f receives the temporary address and produces an unsigned 32-bit result.

Each call receives access to the temporary storage in the state left by earlier calls. Its initial two-word contents are not guaranteed to survive those calls.

### Comparison and final call

After F-9e5fecb2740f returns, the function reads the unsigned 16-bit field at context offset `0x160`.

It compares that field’s zero-extended numeric value against the complete unsigned 32-bit result. The result is not truncated to 16 bits. Any derived result greater than 65,535 therefore differs from every possible context-field value.

The comparison decision is followed by exactly one call to F-289ec4311c64 with the temporary address:

- Unequal operands lead to that call and then return 1.
- Equal operands lead to that call and then return 0.

The comparison precedes the final call. Changes to the context field during that call do not trigger a new comparison.

These return descriptions require the relevant dependencies to return normally. No handling of exceptions, traps, or nonlocal exits is visible.

## Side effects

### Direct accesses

On their respective reachable paths, the function directly:

- Reads the context fields listed above.
- Reads selector storage A for the first lookup.
- Reads selector storage B for the second lookup.
- Writes the two temporary words in payload-then-companion order.
- Reads one 32-bit word from a non-null first lookup result.

No direct write to context memory, either selector location, or the first lookup result is visible.

The global selector reads are conditional effects. Neither occurs when an entry gate fails. The selector-A read occurs before the first lookup call; the selector-B read occurs only after the first lookup, threshold, and predicate gates succeed.

Within either lookup’s argument preparation, the ordering among its two context reads and its selector read remains unspecified by the supplied representation.

### Dependency-mediated effects

The dependencies may affect temporary storage, global selector storage, context fields, resources, or other shared state. Their implementations and preservation guarantees are unavailable.

Accordingly:

- The two lookups may receive different values from the same context offsets.
- Selector storage B may be changed by the first lookup or predicate before it is read.
- Distinct selector locations may hold equal values.
- The comparison observes the `0x160` field after all preceding processing calls.
- No dependency is assumed to preserve globals or acquire, transfer, or release ownership.

### Lifetime and ownership

The temporary storage is local to the invocation and is not returned. Four dependencies receive its address on the completed processing path.

The function does not establish whether those dependencies retain the address, allocate associated resources, or require additional storage. Safe retention beyond the invocation is not established.

F-289ec4311c64 occupies a finalization-shaped position, but this does not prove destructor or release semantics. It is not called on paths that stop at the first lookup, threshold, or predicate gates, even though temporary initialization has already occurred.

Neither lookup result has an established ownership-transfer or release contract.

## Dependencies

| Opaque function ID | Established interaction |
|---|---|
| F-5710a8628cba | First called with two context-field reads and the value read from selector storage A. Conditionally called again with fresh context-field reads and the value read from selector storage B. The first result is null-checked and conditionally dereferenced; the second is forwarded unchecked. |
| F-9d9e918de01f | Called after the threshold passes, with the field at `0x1D8` and the original payload. Zero stops processing; nonzero continues. |
| F-7afe554bcf62 | Receives the temporary address and second lookup result. |
| F-abb7a1ff7712 | Receives the temporary address and zero after F-7afe554bcf62. |
| F-9e5fecb2740f | Receives the temporary address and supplies the unsigned 32-bit comparison result. |
| F-289ec4311c64 | Receives the temporary address after the comparison and before either completed-comparison return. |

Only the two lookup results, predicate result, and derived comparison result are visibly consumed. No stronger dependency contracts are supplied.

## Acceptance examples

These are conditional expectations derived from static evidence, not executed tests. Unless otherwise stated, all required memory is accessible, dependencies return normally, and earlier gates succeed.

1. **Disabled flag:** A zero byte at `0x168` produces return 0. Neither temporary initialization, either global selector read, nor any dependency call occurs.

2. **Excluded companion:** With a nonzero flag and companion −1, the function returns 0 before reading either selector location.

3. **Zero quantity:** With the first two entry conditions satisfied, quantity zero produces return 0 without a lookup.

4. **Null first result:** The first lookup receives the value read from selector storage A and returns null. The function returns 0 without dereferencing the result, reading selector storage B, or calling the predicate.

5. **Zero first word:** A non-null first result whose first 32-bit word is zero produces return 0. No quantity-dependent predicate or temporary-processing call follows.

6. **Below threshold:** Quantity 15 or −15 causes return 0 after a successful first lookup. The selector-A read has occurred; the selector-B read and predicate call have not.

7. **Threshold boundary:** Quantity 16 or −16 reaches F-9d9e918de01f. If that dependency returns zero, the function returns 0 without the second lookup.

8. **Selector mutation between calls:** Suppose selector storage B initially contains one value, and the first lookup changes it to another. If later gates pass and no subsequent change intervenes, the second lookup receives the changed value. It does not use an entry-time snapshot.

9. **Equal selector values:** If the two distinct selector locations hold equal values when read, both lookups may receive the same selector value. Equality of those values does not merge the two reads or the two calls.

10. **Equal comparison:** A derived result of 42 and subsequent context-field read of 42 cause one final call to F-289ec4311c64, followed by return 0.

11. **Unequal comparison:** A derived result of 43 and subsequent context-field read of 42 cause one final call, followed by return 1.

12. **No 16-bit truncation:** A derived result of 65,536 and context-field value zero produce return 1 after the final call, despite equal low 16 bits.

13. **Late context change:** If a processing dependency changes `0x160` before the comparison read, that later value participates in the comparison. A change during the final call does not revise the already selected outcome.

14. **Null second result:** A null second lookup result is passed to F-7afe554bcf62. Whether execution completes is dependency-defined and remains **documented_only**.

15. **Minimum signed quantity:** For −2,147,483,648, the contract does not assert that negation produces a valid positive magnitude or that processing reaches the predicate.

## Uncertainties

The following remain **documented_only**:

- Independently verified calling convention and machine-level type details.
- Selector-storage declarations, exact selector read widths, stored values, and lookup semantics.
- Relative evaluation order of the selector read and the two context reads forming each lookup’s arguments.
- Dependency mutation of either selector location, context memory, temporary storage, or other shared state.
- Semantic field names and ownership rules for context data and lookup results.
- Dependency allocation, retention, release, failure, and synchronization behavior.
- The minimum signed 32-bit quantity’s negation outcome.
- Dependency requirements beyond the initialized eight-byte temporary region.
- Behavior under invalid addresses, inaccessible memory, concurrent mutation, or abnormal dependency exits.
- Whether the final dependency performs destruction, release, or another operation.

Unspecified argument-read order is not an atomic-snapshot guarantee. Nor does the absence of direct selector writes imply immutable selector values.

The staged temporary operations suggest preparation, derivation, and finalization. That is inferred intent, not an established resource-management contract.

## Evidence

The supplied static representation establishes:

- Three short-circuited entry gates.
- Ordered initialization of two 32-bit temporary words.
- Two distinct global selector storage locations, each read when its corresponding lookup is reached.
- No established immutability or inequality of their stored selector values.
- A null-checked first lookup followed by a conditional 32-bit dereference.
- Signed negation followed by a signed threshold test against 16.
- A zero/nonzero predicate gate.
- A second lookup using fresh context reads and its own global selector read.
- Ordered temporary-processing calls.
- A late unsigned 16-bit read compared with a full unsigned 32-bit result.
- One final dependency call on either completed comparison branch.
- Normal return values limited to 0 and 1.

No implementation was executed and no runtime tests were performed. Dependency implementations were not supplied. The acceptance examples describe conditional behavior supported by the static representation, not observations of a running system.
