<!-- Trusted provenance: F-d7d7f1f87df6; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-d7d7f1f87df6

## Purpose

F-d7d7f1f87df6 accepts one of six record selectors, computes a record address using a 20-byte stride, invokes dependencies, conditionally processes the selected record, and performs two ordered one-byte writes.

Its normal result is:

- **0** for selectors 0 through 5, provided the accepted path completes normally.
- **4,294,967,295**, the 32-bit all-ones value, for every other unsigned 32-bit selector.

The established contract concerns argument forwarding, address selection, call order, a byte-level condition, and two explicit byte stores. The supplied representation does not establish the internal behavior of any dependency.

A fixed-size transfer and paired synchronization are plausible interpretations of some calls, not established semantics. In particular, completion with result zero does not certify successful data transfer, continuing storage lifetime, or particular memory contents at return.

## Interface

### Inputs

| Input | Representation | Established use |
|---|---|---|
| Selector | Unsigned 32-bit integer | Accepted exactly when less than 6; selects a displacement of 20 times its value. |
| Auxiliary value | Opaque 32-bit quantity | Passed unchanged as the second argument to F-unknown-02. |

The auxiliary value is not directly dereferenced or validated by this function. It could represent an address, handle, integer, or other encoded value. Its meaning and the requirements imposed by F-unknown-02 are **documented_only**.

The selector comparison is unsigned. A bit pattern that a signed consumer would interpret as negative does not thereby qualify for the accepted path.

### Result

| Path | Normal 32-bit result |
|---|---|
| Selector in 0–5; accepted path completes | 0 |
| Selector in 6–4,294,967,295 | 4,294,967,295 |

A signed consumer may display the rejected-path result as −1. The established result is the all-ones 32-bit pattern; no named error category is supplied.

The function does not use dependency return values to select its result. Zero means that control reached the accepted path’s normal return, not that the dependencies reported success.

### Referenced storage

The function reads a global base value after its initial dependency returns. It then computes a selected address using a 20-byte stride.

| Offset from selected address | Direct access |
|---|---|
| 2 | Read one byte and test whether it is zero |
| 11 | Write the one-byte value 1 |
| 14 | Subsequently write the one-byte value 1 |

The third argument to F-unknown-02 is also 20. Neither that argument nor the stride establishes that the dependency copies a 20-byte record.

No semantic field names, overall allocation extent, or record alignment requirement are established. Direct accesses reach offset 14, but dependency requirements may extend beyond those directly accessed bytes.

### Preconditions for defined normal-path use

For the accepted path to complete with meaningful memory accesses:

- The global base must be readable when consulted.
- The selected storage must be valid for the byte read and each byte write when those operations occur.
- The arguments and execution environment must satisfy every invoked dependency’s requirements.
- Each invoked dependency must return normally.

These are usage requirements, not checks performed by the function. There is no explicit validation of nullness, allocation bounds, permissions, auxiliary-value validity, or storage lifetime.

Storage validity at the direct accesses does not establish validity through return. Any claim about marker values at return additionally requires continuing storage lifetime and appropriate exclusion of intervening modifications.

## Behavior

### Rejected selectors

The selector check precedes every dependency invocation and record-related access.

For an unsigned selector of 6 or greater, the function returns the 32-bit all-ones value without:

- Calling any dependency.
- Reading the global record base.
- Selecting or accessing a record.
- Performing either marker write.

The auxiliary value is not treated as an address on this path. There is no fallback selection, clamping, or reduction into the accepted range.

### Accepted-path sequence

For a selector from 0 through 5, the visible sequence is:

1. **Initial invocation.** F-unknown-01 receives the numeric argument 4.

2. **Base observation and selection.** After that call returns, the function reads the global base and computes the selected address as that base plus 20 times the selector.

3. **Three-argument invocation.** F-unknown-02 receives the selected address, the unchanged auxiliary value, and the numeric argument 20. No returned value is used to determine subsequent behavior.

4. **Conditional-byte read.** After F-unknown-02 returns, the function reads the byte at selected-address offset 2 and tests it against zero.

5. **Conditional invocation.** If the byte is nonzero, F-847a2a401182 receives the selected address. If it is zero, that call is skipped. The condition is not visibly read again.

6. **First marker store.** The function writes the one-byte value 1 at selected-address offset 11.

7. **Second marker store.** The function writes the one-byte value 1 at selected-address offset 14.

8. **Post-store invocation.** F-db50b8e82ff9 receives the numeric argument 0.

9. **Final invocation.** F-unknown-03 receives the numeric argument 4.

10. **Return.** The function returns the 32-bit value zero.

This is the ordering established by the supplied representation. It does not establish atomic execution, memory barriers, hardware-level publication order, or visibility guarantees to concurrent observers.

### Conditional-byte semantics

The decision is zero versus nonzero. Every nonzero byte representation, including 128 and 255, takes the conditional-call path. Interpreting the byte as signed rather than unsigned does not change that decision.

The deciding byte is read after F-unknown-02 returns. An entry-time byte value does not determine the branch if the dependency or another actor changes it before inspection.

If F-847a2a401182 returns normally, both marker writes follow regardless of changes it makes to the condition byte. There is no loop, second condition test, or dependency-result check.

### Address retention and global changes

The global base is observed after F-unknown-01 returns. A base change made by that dependency can therefore affect selection.

Once selected, the address is retained for:

- F-unknown-02’s first argument.
- The offset-2 byte read.
- F-847a2a401182’s argument, when invoked.
- Both marker stores.

No later global-base reload is visible. A subsequent global-base change does not redirect these operations to a newly based record.

Retaining an address does not retain ownership or extend storage lifetime. If a dependency or another actor invalidates the selected storage, the remaining accesses have no established valid outcome.

### Widths and arithmetic

The unsigned selector check occurs before multiplication. Accepted selectors produce displacements of exactly 0, 20, 40, 60, 80, or 100 bytes. This multiplication cannot overflow 32 bits on an accepted path.

The supplied representation uses a 32-bit address-sized value for the selected address. Neither the base-plus-displacement calculation nor the subsequent offset calculations have an explicit range or overflow check.

The contract does not establish usable storage at a wrapped or otherwise invalid address. It also does not establish a portable language-level rule for address overflow beyond the supplied representation.

Both marker stores are one byte wide. They are not 32-bit writes, and their explicit write footprints do not include adjacent bytes. Dependency effects are not similarly bounded.

### Errors and interrupted execution

The rejected selector is the only explicit error-like result. There is no visible rollback, retry, exception handling, or cleanup branch.

A dependency result, if any, is not inspected. If a dependency returns normally after reporting failure through a return value, the visible sequence continues.

If a call fails to return or a memory access faults, subsequent operations are not guaranteed. For example:

- Failure before the condition read can prevent both marker stores.
- Failure in the conditional dependency can prevent both stores.
- Failure after the offset-11 store can leave only that direct store completed.
- Failure in either final dependency can occur after both stores but before a normal return.

No guarantee is made that a resource acquired by an earlier dependency is released after interruption.

## Side effects

### Direct effects

On an accepted path that reaches each operation, the function directly:

- Reads the global base once at the specified point.
- Reads the byte at selected-address offset 2.
- Writes 1 to selected-address offset 11.
- Then writes 1 to selected-address offset 14.

Both writes occur whether the condition byte was zero or nonzero, provided execution reaches them with valid storage.

The function’s direct stores establish write events, not unconditional final-state guarantees. Later dependencies, concurrent actors, or storage-lifetime changes can prevent the stored values from being meaningfully observable at return.

### Dependency-mediated effects

The visible body provides no general frame condition for dependency effects. It does not establish that any dependency preserves:

- The global base or unrelated global state.
- Selected-record contents.
- The lifetime or accessibility of selected storage.
- Caller-supplied storage.
- Synchronization state or resource ownership.

F-db50b8e82ff9 and F-unknown-03 execute after both marker stores. Either could alter the bytes, release the storage, or otherwise affect its subsequent accessibility. Even if neither changes those bytes, another actor could do so unless external synchronization excludes that interference.

### Ownership

No allocation, release, or ownership-transfer operation is explicitly identified in this function.

Passing an address or opaque value to a dependency does not establish copying, retention, consumption, ownership transfer, or release. Those behaviors remain dependent on unavailable contracts.

The function itself supplies no demonstrated lifetime protection for the selected address between calls or through return.

## Dependencies

The three F-unknown labels are document-local opaque IDs for dependencies without supplied published identities.

| Opaque function ID | Arguments | Established invocation condition and position |
|---|---|---|
| F-unknown-01 | 4 | First call on every accepted path; before reading the global base |
| F-unknown-02 | Selected address, auxiliary value, 20 | After address selection; before the condition-byte read |
| F-847a2a401182 | Selected address | Exactly when the inspected byte is nonzero |
| F-db50b8e82ff9 | 0 | After both direct marker stores |
| F-unknown-03 | 4 | After F-db50b8e82ff9; before returning zero |

No dependency return value is used by this function.

**Inferred intent:** The three-argument call resembles a fixed-size data operation. The initial and final calls with argument 4 may bracket record processing.

**Not established:** Transfer direction or extent, overlap behavior, synchronization semantics, resource identity, pairing requirements, storage retention, and error conventions. Matching argument values alone do not prove that the initial and final calls acquire and release the same resource.

## Acceptance examples

These are conditional criteria derived from static evidence, not runtime observations. Accepted-path examples assume the required accesses are valid when performed and invoked dependencies return normally unless otherwise stated.

### Rejected boundary and unsigned interpretation

With selector 6, the result must be the 32-bit all-ones value, with no dependency calls, base read, or marker stores.

The same holds for selector 4,294,967,295. Its possible signed interpretation as −1 must not make it pass the unsigned bound.

### First and last accepted records

With selector 0, selection must use the base observed after F-unknown-01 returns, with zero displacement.

With selector 5, selection must use that observed base plus 100 bytes, subject to the stated address-validity requirements.

In both cases, F-unknown-02 must receive 20 as its third argument.

### Zero and nonzero condition bytes

If the byte at selected-address offset 2 is zero when inspected, F-847a2a401182 must be skipped.

If that byte is 128, F-847a2a401182 must be invoked once with the selected address before either marker store.

In either case, execution that reaches the stores must write offset 11 before offset 14, followed by the two final dependency calls in their specified order.

### Changes before inspection

If the condition byte is initially zero but F-unknown-02 makes it nonzero before inspection, the conditional dependency must be invoked.

If the byte is initially nonzero but becomes zero before inspection, the conditional dependency must be skipped.

The relevant value is the one actually read after F-unknown-02 returns.

### Global-base changes

If F-unknown-01 changes the global base before returning, selection must use the changed value.

If F-unknown-02 subsequently changes the global base again, the condition read, conditional argument, and marker stores must still use the previously selected address. This criterion does not assume that the old storage remains valid; valid subsequent accesses are a separate precondition.

### Ordered stores versus values at return

Without further assumptions, the marker guarantee is limited to the two ordered one-byte stores when execution reaches them with valid storage.

A stronger final-value criterion applies only if:

- The selected storage remains valid and retains the same relevant storage identity through return.
- The stores complete normally.
- Neither final dependency modifies the marker bytes after their stores.
- No concurrent or other actor intervenes with a modification to either marker byte after its store.
- Both final dependencies return normally.

Under those conditions, offsets 11 and 14 contain 1 at normal return.

Normal return plus non-modification by the final dependencies alone is insufficient: a dependency could end the storage lifetime without changing those bytes, and another actor could modify them.

## Uncertainties

The following remain **documented_only**:

- Meaning, provenance, and validity range of the global base.
- Allocation extent, layout semantics, and lifetime of the apparent record region.
- Meanings of bytes at offsets 2, 11, and 14.
- Interpretation and validity requirements of the auxiliary input.
- Full contracts and side effects of every dependency.
- Whether F-unknown-02 transfers data at all.
- Transfer direction, overlap behavior, and aliasing requirements.
- Ownership, retention, allocation, and release behavior.
- Synchronization, reentrancy, and concurrent-access safety.
- Dependency failure reporting and nonlocal exits.
- Environmental consequences of invalid or overflowing addresses.
- Storage validity and memory contents at normal return without additional preservation conditions.

These unknowns are not resolved by familiar call shapes, matching constants, or the fact that the function returns zero.

## Evidence

**Established static evidence:** The supplied representation shows an unsigned comparison against 6, a 20-byte selection stride, an initial call before reading the global base, a three-argument call, a zero/nonzero byte test at offset 2, one conditional dependency call, ordered one-byte stores at offsets 11 and 14, two subsequent calls, and the stated 32-bit normal results.

**Inferred intent:** Fixed-stride record updating, optional record processing, and possible bracketing of the operation by synchronization-related calls.

**Unknown:** Dependency internals, semantic field meanings, ownership and lifetime guarantees, concurrency guarantees, and behavior outside the visible operations.

No runtime tests have been performed.
