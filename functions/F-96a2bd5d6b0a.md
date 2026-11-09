<!-- Trusted provenance: F-96a2bd5d6b0a; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-96a2bd5d6b0a

## Purpose

F-96a2bd5d6b0a performs staged initialization through several dependencies, records a value obtained through shared global state, selects one of two numeric pairs, optionally associates the resulting object with an external context, and returns a derived address.

**Established static behavior:** dependency return values determine successive initialization bases. The returned address is the last initialization base minus `0x110` bytes. Initialization includes separate reads of a shared global base: the first supplies the saved counter-writeback base, while the second locates the counter value to read. These reads do not unconditionally identify the same counter location.

**Inferred intent:** the routine resembles construction of an object containing nested components, an identifier, dimensions, and optional context registration. Those semantic interpretations do not establish field types, ownership, or lifecycle rules.

No explicit allocation, release, rollback, or local error reporting appears in the supplied body.

## Interface

### Inputs

| Input | Representation | Visible use |
|---|---|---|
| Destination | Address supporting a 32-bit write | Receives an initial zero; destination plus `0x04` is passed to the first dependency. |
| Context token | 32-bit value | Stored in the final region before a later comparison. |
| Payload | 32-bit value | Stored during intermediate initialization. |
| Selector | 32-bit value | Copied into an addressable temporary for a dependency; its low 16 bits are stored separately. |
| Configuration | 32-bit value | Stored during intermediate initialization. |
| Option byte | 8-bit value | Stored unchanged during intermediate initialization. |

No explicit validation is performed.

F-ce27efc5fcd6 receives the address of a temporary initially containing the full selector. A change to that temporary does not change the separately supplied selector used for the later 16-bit store. Whether the dependency retains the temporary’s address is unknown; no lifetime beyond this invocation is established.

### Output and address terminology

The output is a 32-bit address-like result:

**result base = final base − `0x110` bytes.**

The following labels distinguish addresses without assuming address-preserving dependencies:

| Label | Definition |
|---|---|
| Destination base | First input. |
| First base | Result of F-e194f3fbdfaa. |
| Second base | Result of F-ce27efc5fcd6. |
| Third base | Result of F-5b24edfb22c0. |
| Fourth base | Result of F-ff4dff756de0. |
| Final base | Result of F-f8dea028dcbd. |
| Result base | Final base minus `0x110`. |

All offsets are in bytes. Final-base-relative accesses are also described relative to the result base where the arithmetic is representable. No dependency’s result is assumed equal to its input.

There is no explicit success flag or failure sentinel. The function does not translate dependency failures or invalid accesses into a local error result.

### Preconditions

Normal completion requires:

- Every directly accessed address to support the indicated access width.
- Returned bases to support the accesses subsequently made through them.
- Dependencies to accept their supplied arguments.
- Both global-base reads, the counter read, and the saved-base writeback to reach usable storage.
- Any optional context dereferences to reach readable storage.
- Address calculations and the counter increment to have valid behavior in the applicable execution model.

Allocation extent, alignment, valid token ranges, and dependency-specific requirements remain **documented_only**. The visible accesses do not establish a complete object layout.

## Behavior

### 1. Begin initialization

The first direct write is a 32-bit zero at the destination base.

The selector is then copied into a 32-bit temporary. F-e194f3fbdfaa receives destination base plus `0x04`. F-ce27efc5fcd6 subsequently receives first base plus `0x10` and the temporary’s address.

The latter dependency’s returned address becomes the second base. Subsequent initialization uses that returned address, not an assumed address within the original destination.

### 2. Initialize the second-base region

Direct writes occur in this order:

| Offset from second base | Width | Value |
|---|---:|---:|
| `0x1C` | 32 bits | `0` |
| `0x24` | 32 bits | `0x80` |
| `0x20` | 32 bits | `0` |
| `0x28` | 8 bits | `1` |
| `0x29` | 8 bits | `1` |
| `0x2A` | 8 bits | `0` |
| `0x2B` | 8 bits | `0xFF` |

F-5b24edfb22c0 then receives second base plus `0x2C`. Its result becomes the third base.

### 3. Initialize the third-base region

Direct writes occur in this order:

| Offset from third base | Width | Value |
|---|---:|---|
| `0x08` | 32 bits | `0` |
| `0x0C` | 32 bits | `0` |
| `0x10` | 32 bits | `0` |
| `0x14` | 32 bits | `0` |
| `0x18` | 32 bits | `0` |
| `0x1C` | 32 bits | `0` |
| `0x20` | 8 bits | `1` |
| `0x22` | 16 bits | Low 16 bits of selector |
| `0x28` | 32 bits | `0` |
| `0x24` | 32 bits | Configuration |
| `0x2C` | 32 bits | `0` |
| `0x34` | 32 bits | `0` |
| `0x38` | 32 bits | Payload |

The selector store supplies a 16-bit representation, not a range check or saturation. Its interpretation by subsequent consumers is unknown.

F-ff4dff756de0 then receives third base plus `0x3C`. Its result becomes the fourth base.

### 4. Initialize the fourth-base region

Direct writes occur in this order:

| Offset from fourth base | Width | Value |
|---|---:|---|
| `0x58` | 32 bits | `0` |
| `0x5C` | 32 bits | `0` |
| `0x70` | 8 bits | `0` |
| `0x71` | 8 bits | `0` |
| `0x72` | 8 bits | `1` |
| `0x73` | 8 bits | Option byte |
| `0x74` | 8 bits | `0` |
| `0x80` | 32 bits | `0xFFFFFFFF` |
| `0x84` | 32 bits | `0xFFFFFFFF` |
| `0x88` | 32 bits | `0` |
| `0x8C` | 32 bits | `0` |
| `0x90` | 32 bits | `0` |

The all-ones values are established bit patterns. Their possible use as signed sentinels is inferred.

F-f8dea028dcbd then receives fourth base plus `0x94`. Its result becomes the final base.

### 5. Capture a counter value and establish the comparison byte

The shared-state access sequence is significant:

1. Read the shared global base and retain that address for the eventual counter writeback.
2. Derive the result base as final base minus `0x110`.
3. Read the shared global base again.
4. Read a 32-bit counter value at the second observed base plus `0x04`.
5. Write the captured counter value to final base plus `0x1C`, equivalently result base plus `0x12C`.
6. Write the captured value plus one to the **first, saved base plus `0x04`**.

The writeback does not reload the global base and does not reread the destination counter. The two counter locations coincide if the two global-base reads agree, assuming ordinary address arithmetic. Equality of those reads is not guaranteed by the supplied body. If they differ, a value read through the second base is incremented and written through the first.

There is no dependency call between these two global-base reads, but the source still contains two distinct reads. No atomic snapshot or synchronized increment is established.

The increment uses a signed integer value in the supplied representation. A defined wrap from the maximum signed value is not established; overflow behavior remains **documented_only**.

The routine next:

- Writes the context token at final base minus `0x1C`, equivalently result base plus `0xF4`.
- Calls F-bb50a072e902 with zero.
- Reads the 32-bit field at result base plus `0xF4` after that call.
- Writes a one-byte equality result at result base plus `0xF8`: `1` for equality with the dependency result, otherwise `0`.

The comparison is against the stored field as read after the dependency, not necessarily against the original context-token argument. Preservation of that field is not established.

### 6. Initialize additional components and select a numeric pair

The following calls occur in order:

| Dependency | Argument |
|---|---|
| F-2d9a83e3d450 | Result base |
| F-ca73cb512206 | Result base plus `0x20` |
| F-ca73cb512206 | Result base plus `0xB4` |
| F-ca73cb512206 | Result base plus `0xA4` |

Next, the function writes a 32-bit zero at result base plus `0xC8`, followed by a 32-bit zero at result base plus `0xC4`.

It then reads the byte at result base plus `0xF8`. That single captured zero/nonzero decision controls both following stores:

| Byte read | Value at result base plus `0xD0` | Value at result base plus `0xCC` |
|---|---:|---:|
| Zero | `0x2D0` — 720 | `0x240` — 576 |
| Nonzero | `0x140` — 320 | `0xF0` — 240 |

Both stores are 32 bits wide. The `0xD0` store precedes the `0xCC` store. There is no second decision-byte read between them.

The pair plausibly represents dimensions; orientation and units are unknown.

The function then calls:

1. F-b00260d1087a with result base plus `0xD4`.
2. F-ca73cb512206 with result base plus `0xDC`.
3. F-df6da5bd408b with the result base and zero.

These dependencies are not assumed to preserve the previously written fields.

### 7. Perform optional association

After F-df6da5bd408b returns, the function reads the byte at result base plus `0xEF`.

If that byte is zero, association is skipped. Otherwise, the function reads the byte at result base plus `0xF8` again:

- **Zero:** call F-938c950e5046, then read a 32-bit address-like field at offset `0x2C` from its returned address. That field supplies the context address.
- **Nonzero:** call F-38aa1bb9efd0 and use its result directly as the context address.

The function reads a 32-bit field at context address plus `0xE8`, then calls F-627c5664a32c with that field value and the result base. Its return value is not used.

No null check is visible before either context dereference.

The association decision reads `0xF8` separately from dimension selection. Intervening dependencies can change the byte; the two decisions need not agree.

### 8. Finish

On either normally completing association path, including the skipped path, the function:

1. Calls F-8157a180d901 with `8`.
2. Calls F-6db2f626878d with no explicit arguments.
3. Calls F-200542792196 with result base plus `0x7C` and the value returned by F-6db2f626878d.
4. Returns the result base.

No final status check is visible.

## Side effects

### Direct memory effects

The function performs the listed writes incrementally. It does not clear complete regions. Unlisted bytes cannot be classified as zero, preserved, padding, or initialized solely from this body.

Shared-state effects include two reads of the global base, a counter read through the second observed base, and a counter write through the first saved base. Calling this unconditionally an increment of one shared counter location would omit a material address-selection distinction.

The stored counter value is captured before the object-field write. The later writeback uses that captured value and saved address even if intervening aliasing affects the global-base storage or counter memory.

Supplied and returned addresses are not declared non-aliasing. Overlapping regions can make earlier writes affect later reads. The described values characterize the direct operations when reached, not guaranteed final contents after all dependencies have returned.

### Ownership, lifetime, and failures

No explicit allocation or deallocation appears in this body. Dependencies may nevertheless allocate, release, retain addresses, mutate globals, or perform unrelated effects; their implementations are unavailable.

Ownership of the result, retention by association, eventual release obligations, and legality of repeated initialization remain unknown.

There is no visible rollback. An invalid access or abnormal dependency exit can leave earlier writes and any completed counter writeback in place. The routine provides no explicit recovery path or local failure translation.

## Dependencies

Only opaque IDs establish dependency identity.

| Dependency | Visible contract at this call site |
|---|---|
| F-e194f3fbdfaa | Receives destination plus `0x04`; returns first base. |
| F-ce27efc5fcd6 | Receives first base plus `0x10` and selector temporary’s address; returns second base. |
| F-5b24edfb22c0 | Receives second base plus `0x2C`; returns third base. |
| F-ff4dff756de0 | Receives third base plus `0x3C`; returns fourth base. |
| F-f8dea028dcbd | Receives fourth base plus `0x94`; returns final base. |
| F-bb50a072e902 | Receives zero; returns the comparison operand. |
| F-2d9a83e3d450 | Receives result base before dimension selection. |
| F-ca73cb512206 | Receives result-relative addresses `0x20`, `0xB4`, `0xA4`, and later `0xDC`, in that order. |
| F-b00260d1087a | Receives result base plus `0xD4`. |
| F-df6da5bd408b | Receives result base and zero before association eligibility is read. |
| F-938c950e5046 | Returns an address used for the additional `0x2C` indirection. |
| F-38aa1bb9efd0 | Returns the alternate context address directly. |
| F-627c5664a32c | Receives the selected context’s `0xE8` field and result base. |
| F-8157a180d901 | Receives `8` after association or its omission. |
| F-6db2f626878d | Supplies the second argument for the final dependency. |
| F-200542792196 | Receives result base plus `0x7C` and the preceding dependency result. |

No dependency is assumed to preserve globals or object fields, or to acquire or release ownership.

## Acceptance examples

These are conditional static expectations, not executed tests. Each assumes valid accesses and normal return from the dependencies needed to reach the stated operation.

1. **Selector truncation.** For selector `0x12345678`, F-ce27efc5fcd6 receives a temporary initially containing that full value. The later direct 16-bit store at third base plus `0x22` writes `0x5678`, independently of changes to the temporary.

2. **Matching global-base reads.** If both global-base reads yield the same address and its counter read yields `41`, the function writes `41` at final base plus `0x1C`, then writes `42` at that shared base plus `0x04`. The read and write target the same counter location under this condition.

3. **Different global-base reads.** If the first read yields one valid base, the second yields a distinct base with a distinct counter location, and the second location contains `41`, the object receives `41` and the first location receives `42`. The first location’s previous value does not determine the incremented value. The second location is not the direct writeback target.

4. **Signed boundary.** A captured counter value of `0x7FFFFFFE` produces a writeback value of `0x7FFFFFFF`. A captured maximum signed 32-bit value does not establish a required wrap to `0x80000000`; that boundary remains **documented_only**.

5. **Equal comparison.** If F-bb50a072e902 returns the value subsequently read at result base plus `0xF4`, the equality store writes `1` at `0xF8`. If the byte remains nonzero until dimension selection, the pair written is 320 at `0xD0`, then 240 at `0xCC`.

6. **Intervening decision mutation.** If the equality store writes `1` but an intervening dependency changes `0xF8` to zero before dimension selection, the pair written is 720, then 576. The earlier comparison does not override the later read.

7. **Association disabled.** If `0xEF` is zero when read after F-df6da5bd408b, neither context resolver nor F-627c5664a32c is called. The final three dependency calls still occur.

8. **Association through indirection.** If `0xEF` is nonzero and the subsequent `0xF8` read is zero, the context address comes from F-938c950e5046’s returned address plus `0x2C`. The selected context’s `0xE8` field becomes the first association argument.

9. **Independent later decision.** If dimension selection reads zero but the later association decision reads nonzero, the larger numeric pair is written while association uses F-38aa1bb9efd0.

10. **Returned-address relationship.** If all five chained initialization dependencies return exactly their first supplied addresses, the accumulated offsets are `0x04 + 0x10 + 0x2C + 0x3C + 0x94 = 0x110`; the result base therefore equals the destination. Without that condition, equality is not established.

## Uncertainties

The following remain **documented_only**:

- Exact object type, allocation size, byte order, alignment, and field meanings.
- Dependency return-address guarantees and failure conventions.
- Whether the two global-base reads agree in any particular execution, and what synchronization governs them.
- Counter overflow semantics, atomicity, thread safety, and reentrancy.
- Address-arithmetic behavior at representation boundaries.
- Whether the option-byte store corresponds to result offset `0xEF`. This follows under the address-preserving chain condition, but not unconditionally.
- Initialization or preservation of unlisted fields.
- Ownership, retention, cleanup, repeated-initialization rules, and recovery obligations.
- Whether dependencies terminate execution or report errors outside their return values.

No unknown is replaced with an assumed success condition, memory-preservation guarantee, or fallback.

## Evidence

This contract derives from static inspection of the supplied function body and opaque dependency mapping.

Established evidence includes access widths, literal values, relative offsets, ordered direct accesses, dependency order, return-value use, branch conditions, and the final address calculation. In particular, the body separately reads the shared global base for saved writeback and for locating the counter read; the writeback uses the first saved base.

Constructor-like purpose, dimension semantics, registration intent, and sentinel meanings are inferences rather than established guarantees.

Dependency implementations were not supplied. No runtime tests, memory observations, or dynamic validation have been performed.
