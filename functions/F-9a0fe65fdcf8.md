<!-- Trusted provenance: F-9a0fe65fdcf8; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-9a0fe65fdcf8

## Purpose

F-9a0fe65fdcf8 performs a state-dependent operation on a caller-supplied memory region. It first replaces the region’s leading 32-bit word with a value read from external storage. It then examines a byte at offset 24:

- A zero byte selects a path that invokes F-5de1ab857c48 on the address at offset 28 and adjusts that dependency’s integer result by subtracting 28.
- A nonzero byte selects a path that invokes F-1dc7fdb390e6, then F-feee9156a257, using values obtained from the 32-bit word at offset 20. The second dependency is marked as non-returning in the supplied static representation.

These are established structural behaviors. Interpreting the operation as destruction, disposal, reference management, or object conversion would require information not supplied here.

This contract describes the visible operations and their ordering. It does not assign ownership semantics to the dependencies or assume that they preserve the supplied region or external state.

## Interface

### Input

The function accepts one pointer, designated here as **B**, to storage addressed in 4-byte units by the supplied representation.

The directly relevant layout is:

| Byte offset from B | Width | Direct use |
|---|---:|---|
| 0 | 4 bytes | Destination of the initial external-value assignment |
| 20 | 4 bytes | Argument source on the nonzero-byte path |
| 24 | 1 byte | Branch selector |
| 28 | No direct access shown | Address passed to F-5de1ab857c48 on the zero-byte path |

Offsets 4–19 and 25–27 have no direct accesses in the supplied body. This does not establish that they are unused by dependencies.

The address passed at offset 28 is derived from B; it is not loaded from a pointer field stored there. Likewise, the offset-20 argument is a 32-bit value loaded from the region, not the address of that field.

### Preconditions

For the function’s direct operations to be valid:

- B must support a writable 4-byte access at offset 0.
- B must support a readable byte at offset 24.
- On the nonzero-byte path, B must support the required 4-byte reads at offset 20.
- Address formation for B plus 28 must be valid on any path reaching F-5de1ab857c48.
- Alignment and memory-access requirements of the execution environment must be satisfied.

The function contains no visible validation of these conditions. It also performs no visible null check, bounds check, or dependency-availability check.

Additional preconditions imposed by the three dependencies are unknown. In particular, merely providing the directly accessed bytes is not sufficient evidence that a dependency call is safe.

### Output

On the ordinary zero-byte path, assuming F-5de1ab857c48 returns normally, the function returns that dependency’s integer result minus 28.

The recovered interface represents both the dependency result and this function’s result as an integer type. Its exact machine width and overflow behavior are not independently established by the supplied body. For results whose subtraction is representable, the arithmetic relation is unambiguous.

On the nonzero-byte path, the expected control flow does not produce an ordinary result because F-feee9156a257 is marked non-returning. That marking is static evidence, not a demonstrated runtime guarantee.

## Behavior

### Initial state replacement

Before selecting a branch, the function reads a value from external storage and assigns it to the 32-bit word at offset 0 from B.

The supplied body does not establish the external value’s numeric contents or semantic role. It must not be treated as a documented constant, type identifier, dispatch pointer, or ownership marker without further evidence.

The previous contents of the destination word are not directly read by this function before replacement. No restoration of that word is shown.

The external value is obtained for this assignment before either branch dependency is invoked. A later dependency changing the external storage would not change the value already written by this assignment, although dependencies may themselves modify the destination.

### Branch selection

After the leading-word assignment, the function reads one byte at offset 24 from B and compares it with zero.

Only zero versus nonzero matters. No particular nonzero value is distinguished. Whether the byte is interpreted as a signed or unsigned character does not affect this test.

The branch selector is read after the initial write. There is no visible second read of this byte for branch selection. Subsequent dependency changes to it do not, by themselves, cause the visible branch decision to be repeated.

### Zero-byte path

When the selected byte is zero:

1. Neither F-1dc7fdb390e6 nor F-feee9156a257 is directly invoked.
2. The function forms the address 28 bytes beyond B.
3. It passes that address to F-5de1ab857c48.
4. If that dependency returns normally with integer result R, the function computes R minus 28 and returns the computed value.

No direct load from offset 20 occurs on this path. There is also no direct dereference of the address at offset 28 by this function; any access through that address belongs to the dependency’s behavior.

The subtraction is an adjustment of the returned integer. The body does not establish that R is an address, allocation size, offset, or success code. The matching value of 28 in the address displacement and return adjustment suggests a relationship, but its purpose remains inferred rather than established.

### Nonzero-byte path

When the selected byte is nonzero:

1. The function loads the 32-bit word at offset 20 from B.
2. It passes that loaded value to F-1dc7fdb390e6.
3. If F-1dc7fdb390e6 returns normally, the function loads the word at offset 20 again.
4. It passes the newly loaded value to F-feee9156a257.

The two argument expressions are separate reads with a dependency call between them. They must not be specified as a single saved argument reused twice.

If F-1dc7fdb390e6 changes the offset-20 word, the second dependency receives the changed value, provided the second read remains valid. If that dependency invalidates the region, the subsequent read is not protected by any visible check. The contract does not assume that the first dependency preserves B’s storage.

F-1dc7fdb390e6 has no result consumed by the supplied body. This does not mean that it lacks side effects or that its failures are ignored in every possible execution model.

F-feee9156a257 is explicitly marked as not returning. Under that marking, execution does not reach F-5de1ab857c48 or the final arithmetic on this path. The representation contains a subsequent common sequence, but its reachability after an unexpected return from F-feee9156a257 is not established as supported behavior.

### Integer and address arithmetic

The field widths and displacements follow from 4-byte element addressing:

- The argument field is at 5 × 4 = 20 bytes.
- The branch byte is at 6 × 4 = 24 bytes.
- The derived dependency argument is at 7 × 4 = 28 bytes.
- The final arithmetic subtracts the integer value 28.

The argument to F-5de1ab857c48 is therefore not displaced by seven bytes or seven pointer-sized slots of an independently assumed width.

The supplied body provides no explicit overflow check, saturation, or widening around the returned integer subtraction. For boundary results, a specification of modulo wrapping or language-level overflow failure would require additional evidence about the recovered types and actual execution semantics.

## Side effects

The only direct memory write shown is the replacement of the 32-bit word at offset 0. It precedes the branch-byte read and all dependency calls.

The function also directly reads:

- The external value used for the initial replacement.
- The byte at offset 24.
- On the nonzero-byte path, the word at offset 20 before each of the two dependency calls, subject to control flow reaching those reads.

These are source-visible access ordering requirements, not a guarantee about hardware transactions, atomicity, volatility, or cross-thread synchronization.

Dependency side effects are unrestricted by the available evidence. They may affect the supplied region, external storage, or other state. No preservation guarantee is established for bytes that this function does not itself write.

The function visibly performs no allocation, deallocation, reference-count update, lock operation, logging, or error reporting of its own. Such actions may occur inside dependencies. There is no basis for assigning ownership transfer, borrowing duration, or release responsibilities to either argument form.

If a dependency does not return, this function contains no visible recovery or cleanup sequence afterward. If an invalid access occurs, no local error conversion or rollback is shown. The initial write is not conditionally deferred until the rest of the operation succeeds.

## Dependencies

| Opaque function ID | Invocation condition | Argument | Result use and limits |
|---|---|---|---|
| F-1dc7fdb390e6 | The offset-24 byte is nonzero | A 32-bit value read from offset 20 | No result is consumed. Must return normally for the next read and call to occur. Preservation and ownership behavior are unknown. |
| F-feee9156a257 | The nonzero-byte path reaches its second call | A fresh 32-bit read from offset 20 | Marked non-returning. Termination mechanism and memory effects are unknown. |
| F-5de1ab857c48 | The zero-byte path reaches its call | Address B plus 28 bytes | Its integer result is reduced by 28. Argument requirements and side effects are unknown. |

External storage supplying the initial 32-bit value is an additional state dependency. Its initialization rules, mutability, and semantic meaning are undocumented here.

## Acceptance examples

These are conditional acceptance criteria derived from static evidence, not executed tests.

### Zero selector and ordinary result

Given valid storage, a zero byte at offset 24, and F-5de1ab857c48 returning 100:

- The leading word is replaced before the dependency call.
- F-5de1ab857c48 receives B plus 28 bytes.
- Neither nonzero-path dependency is directly called.
- The function returns 72, assuming the represented integer type can express the arithmetic.

### Zero selector and zero adjusted result

If the zero-byte path’s dependency returns 28, the function returns 0.

This example establishes only arithmetic. It does not establish that zero means success.

If that dependency returns 0, the corresponding result is −28 where representable; the body contains no special handling for a zero dependency result.

### Nonzero selector without intervening field modification

Given a nonzero byte at offset 24 and a readable offset-20 word holding value A, if F-1dc7fdb390e6 returns without changing that word or invalidating its storage:

- The first dependency receives A.
- The second read also obtains A.
- F-feee9156a257 receives A.

No ordinary function result is required under the second dependency’s non-returning contract.

### Nonzero selector with intervening field modification

Given the same path, if F-1dc7fdb390e6 changes the offset-20 word from A to B and returns with the storage valid:

- F-1dc7fdb390e6 receives A.
- F-feee9156a257 receives B.

An implementation that always reuses A for the second call would not preserve the visible behavior.

### Nonzero selector with selector modification

If F-1dc7fdb390e6 changes the offset-24 byte to zero and returns normally, that change does not cause the function to reselect the zero-byte path. The next visible operation remains the offset-20 read for F-feee9156a257.

### Invalid storage or arithmetic boundary

A null or insufficiently accessible input has no specified error return from this body.

If the dependency result minus 28 is outside the result type’s representable range, acceptance is **documented_only** until integer width and overflow semantics are established. No wrapping or saturation result is asserted.

## Uncertainties

The following remain **documented_only**:

- The complete object layout and the minimum storage extent required by dependencies.
- The meaning and lifetime of the external value written at offset 0.
- Whether the offset-20 value represents a pointer, handle, identifier, or another 32-bit quantity.
- Ownership, allocation, release, and preservation behavior of every dependency.
- The mechanism by which F-feee9156a257 does not return.
- Supported behavior if that dependency unexpectedly returns.
- Exact machine integer and pointer widths, return-boundary overflow semantics, and ABI details beyond the visible 4-byte field operations.
- Concurrency behavior and any synchronization requirements.

A lifecycle-related purpose and a relationship between the offset-28 address and the return adjustment are plausible interpretations, not contractual facts.

## Evidence

Established static evidence consists solely of the supplied function representation: one leading 32-bit assignment from external storage, one byte-based branch at offset 24, two separately evaluated offset-20 argument reads on the nonzero path, a non-return annotation on its second dependency, and an offset-28 address argument followed by subtraction of 28 on the ordinary return path.

The dependency implementations, callers, external storage definition, and runtime environment were not supplied. Comments and recovered annotations are not independent proof of dependency behavior.

No runtime tests have been performed. The acceptance examples express conditional obligations rather than observed results.
