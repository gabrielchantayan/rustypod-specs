<!-- Trusted provenance: F-dfe6e6b7ec7c; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-dfe6e6b7ec7c

## Purpose

F-dfe6e6b7ec7c conditionally invokes an external operation on two indexed, fixed-stride records, then updates a four-byte field in each record.

The established behavior is narrower than a record-copy or record-transfer API:

- Two signed index arguments are screened in order.
- Equal indices are rejected.
- The source record must have a particular leading byte and a nonzero four-byte field.
- An external dependency receives the destination address, source address, and the value 276.
- After that dependency returns normally, the source’s field at offset 8 is assigned the destination index, and the destination’s field at offset 8 is cleared.
- Explicit rejection returns 9. Completion of the external call and both stores returns 0.

A copy-like operation is a plausible interpretation of the dependency’s arguments, but its behavior is not established. Likewise, the record fields’ business meanings are unknown.

Importantly, the index checks do **not** establish an ordinary nonnegative array range. They admit negative signed indices.

## Interface

### Arguments

| Position | Representation | Established use |
|---|---|---|
| First | Opaque four-byte value | Not read or forwarded by the visible function. |
| Second | Signed 32-bit integer | Source index. |
| Third | Signed 32-bit integer | Destination index. |

The first argument has no visible effect on control flow, address selection, dependency arguments, or explicit stores. No pointer validity requirement can be inferred for it.

### Return value

The function returns a four-byte value:

- **9:** an explicit validation condition rejects the request.
- **0:** all visible validations pass, the dependency returns normally, and both subsequent stores complete.

No other explicit return value is present. A fault, exception, nonlocal exit, or failure inside the dependency is not converted into one of these values by any visible handling.

### Global and record interface

A global value supplies the base address used for record addressing. Records are selected using an index-scaled displacement with a stride of **276 bytes**, or **0x114** bytes.

The visible record layout is:

| Byte offset | Width | Access | Requirement or assignment |
|---|---:|---|---|
| 0 | 1 byte | Source read | Must equal 6. |
| 8 | 4 bytes | Source write after dependency | Receives the destination index’s 32-bit representation. |
| 8 | 4 bytes | Destination write after source write | Receives zero. |
| 12 | 4 bytes | Source read | Must be nonzero. |

These are offsets within the selected records, not declarations of a complete record format. Other bytes are not directly accessed by the visible function, although the dependency may access them.

### Memory preconditions

For defined execution along a path that reaches the relevant operation:

- The global base value must be readable.
- The selected source address must support the one-byte read at offset 0.
- If that byte equals 6, the source must support the four-byte read at offset 12.
- If the dependency returns normally, the saved source and destination addresses must support four-byte writes at offset 8.
- The dependency’s own memory-access and alignment requirements must also be satisfied. Those requirements are unknown.

The index checks alone do not prove any of these conditions. In particular, they neither establish an allocation boundary nor exclude negative displacements.

## Behavior

### 1. Source-index screening

The source index is checked first.

The accepted signed 32-bit values are exactly:

- Every negative value, from −2,147,483,648 through −1.
- Positive values 1 and 2.

Zero and every positive value greater than or equal to 3 are rejected with return value 9.

This follows from the combination of a conditional subtraction, its signed-overflow indication, and a sign comparison. The overflow-sensitive test must not be simplified into a check of only the wrapped subtraction’s sign.

For example, the most negative signed 32-bit value remains accepted even though subtracting 3 from it overflows a signed 32-bit representation.

If this screening rejects the source index, no destination-index screening, global-base read, record read, dependency call, or record write is visible.

### 2. Destination-index screening

The destination index is screened by the same rule, but only after the source index passes.

It therefore accepts all negative signed 32-bit values and the values 1 and 2, and rejects zero and positive values of at least 3.

A rejection returns 9 before any visible global-base or record access.

### 3. Equality rejection

If the two accepted indices are equal, the function returns 9.

This is an integer-equality check, not a resolved-address comparison. It does not independently prove that the selected memory regions are distinct or nonoverlapping.

No record or global-base access precedes this equality rejection in the visible execution.

### 4. Source selection and leading-byte validation

The function reads the global base value and derives the source address using the source index and the 276-byte stride.

It reads one byte at source offset 0. If that byte is not 6, it returns 9.

On this path:

- The source field at offset 12 is not read.
- The destination address is not yet derived.
- The dependency is not called.
- No explicit record write occurs.

### 5. Source nonzero-field validation

If the leading byte is 6, the function reads a four-byte integer at source offset 12.

If that value is zero, it returns 9. Any nonzero bit pattern satisfies this check; there is no visible requirement that the value be positive.

The function does not visibly use this field as an address, count, ownership token, or argument to the dependency. Such meanings would be inferred rather than established.

### 6. Destination selection

After both source-field checks pass, the function reads the global base value again and derives the destination address using the destination index and the same 276-byte stride.

Thus source and destination selection use two separate visible accesses to the global base. There is no visible synchronization or snapshot mechanism requiring those accesses to observe the same value.

No destination field is checked before the dependency call. In particular, the function does not directly validate destination state, availability, or contents.

### 7. Dependency invocation

The function invokes **F-dependency-01** with these ordered arguments:

1. The selected destination address.
2. The previously selected source address.
3. The integer value 276.

The dependency’s return value is not inspected.

The addresses are selected before the call and retained for the subsequent writes. The visible function does not reload the global base or recompute either address after the dependency returns.

### 8. Ordered post-call writes

If the dependency returns normally:

1. A four-byte store at source offset 8 writes the destination index.
2. A four-byte store at destination offset 8 writes zero.
3. The function returns 0.

The order is significant when the store locations overlap. The later destination store takes precedence over any bytes it shares with the earlier source store.

The visible function does not recheck the source’s leading byte or offset-12 field after the dependency. Changes to those fields during the call do not introduce another visible rejection condition.

## Side effects

### Explicit effects

Before the dependency call, the function performs only the described validation reads and address calculations.

After the dependency returns normally, it performs exactly the two described four-byte stores. It does not directly clear the source’s leading byte or offset-12 field.

No explicit allocation, release, lock, reference-count adjustment, or global-base assignment is visible.

### Dependency-mediated effects

F-dependency-01 may have additional effects that cannot be determined from this function. In particular, this contract does not assume that it:

- Copies 276 bytes.
- Supports overlapping regions.
- Preserves the global base value.
- Preserves either record.
- Leaves the selected storage allocated.
- Acquires, transfers, or releases ownership.
- Reports failure through its return value.

A normal return from the dependency causes the two stores to be attempted regardless of its returned value.

If the dependency invalidates either saved address, the following stores may access invalid memory. No visible guard prevents that outcome.

### Rejection and partial completion

An explicit return of 9 occurs before the dependency and before either explicit store.

This does not make the overall operation transactional. If the dependency has effects and then fails to return normally, those effects are not visibly rolled back. Similarly, if execution fails between the two stores, the source store may already have completed without the destination store completing.

No atomicity or concurrency guarantee is established.

## Dependencies

### F-dependency-01

This is a document-local opaque identifier for the sole external callee visible in the supplied material. Its independently established identity and implementation are unavailable.

**Established call contract at this boundary:**

- Receives destination address first.
- Receives source address second.
- Receives 276 third.
- Is called once on the fully validated path.
- Is not called on any explicit rejection path.
- Has no return value used by F-dfe6e6b7ec7c.
- Must return normally for the visible post-call stores and return value 0 to occur.

**Inferred intent:**

The address ordering and byte-sized record stride suggest an operation over one record, possibly copying from source to destination.

**Unknown:**

Its actual reads, writes, overlap rules, argument widths beyond the supplied representation, ownership rules, global effects, and failure behavior.

There is no visible recursive call to F-dfe6e6b7ec7c.

## Acceptance examples

These are conditional contract examples derived from static evidence, not executed tests.

### Index rejection without record access

- Source index 0, with any destination index: returns 9 before any visible global-base or record access.
- Source index 3, destination index 1: returns 9 at source-index screening.
- Source index 1, destination index 0: returns 9 at destination-index screening.
- Source index 2, destination index 2: both range screenings pass, but equality causes return value 9 before record access.

### Negative indices remain eligible

With source index −1 and destination index 1, both index screenings and the equality check pass.

Subsequent behavior depends on the selected memory. This does not imply that memory before the nominal base is valid. If the selected source cannot be read, no normal return is guaranteed.

The same index-screening conclusion applies to source index −2,147,483,648 and destination index 1. Signed subtraction overflow does not turn that source index into an explicit rejection.

### Leading-byte rejection

Assume indices 1 and 2 and a readable selected source whose byte at offset 0 is 5.

The function returns 9 without reading source offset 12, invoking F-dependency-01, or performing either explicit store.

### Zero-field rejection

Assume distinct accepted indices, a source leading byte of 6, and a four-byte zero at source offset 12.

The function returns 9. The destination address is not yet selected, and the dependency is not called.

### Normal completion with distinct, valid records

Assume:

- Source index 1 and destination index 2.
- Both base reads select the expected valid record storage.
- The source leading byte is 6.
- The source field at offset 12 is nonzero.
- F-dependency-01 returns normally and leaves both saved addresses writable.
- The two offset-8 store locations do not overlap.

The dependency receives destination address, source address, and 276. After it returns, source offset 8 becomes 2 and destination offset 8 becomes zero. The function returns 0.

No conclusion about the remaining record bytes follows without the dependency’s contract.

### Dependency return value is ignored

Under otherwise valid completion conditions, changing only the dependency’s returned value does not change the visible control flow. The two stores still occur and the function returns 0.

This example does not cover a dependency that fails to return normally.

### Coincident store addresses

If unequal accepted indices nevertheless resolve to the same offset-8 address, and all accesses and the dependency complete normally, the destination’s later zero store determines the final four bytes at that location.

Whether such address coincidence is possible depends on the address-arithmetic model and base observations; unequal indices alone do not exclude it.

## Uncertainties

The following are **documented_only**, not assumptions supplied to complete the contract:

- **Address arithmetic:** The evidence uses 32-bit integer operations and an integer-like base representation. Pointer width, address-overflow semantics, and the validity of resulting addresses are not independently established.
- **Wrapped addressing:** Under a 32-bit modular address model, distinct index products can coincide because the stride is not relatively prime to the address-space size. Such aliasing is conditional on that model, not a guaranteed runtime property.
- **Endianness and alignment:** Neither is established. The field widths and zero/nonzero tests are established, but byte encoding of nonzero stored indices remains platform-dependent.
- **Global consistency:** No synchronization is visible between the two base reads. Concurrent modification and memory-model guarantees are unknown.
- **Record semantics:** The meanings of the leading byte, offset-8 field, and offset-12 field are unknown.
- **Ownership:** No ownership transfer or lifetime guarantee can be inferred from the visible function or its dependency arguments.
- **External operation:** Copy behavior, overlap safety, memory extent, failure reporting, and resource effects remain unknown.
- **Fault handling:** No recovery is visible for invalid reads, invalid writes, or dependency failures.

## Evidence

This contract is based solely on the supplied static function representation.

Established evidence includes the ordered index checks, signed-overflow-sensitive comparisons, equality rejection, 276-byte addressing stride, source-field reads, external-call argument order, ignored dependency return value, ordered four-byte stores, and explicit return values 9 and 0.

Copy-like intent and record-field meanings are not established facts. Missing callee and platform behavior remain explicitly documented as unknown.

No runtime tests, dependency inspection, or implementation execution have been performed.
