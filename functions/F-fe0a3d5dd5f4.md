<!-- Trusted provenance: F-fe0a3d5dd5f4; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-fe0a3d5dd5f4

## Purpose

F-fe0a3d5dd5f4 conditionally constructs a resource, prepares two groups of temporary numeric pairs, creates and populates two auxiliary objects, and connects their recorded addresses through a 40-byte configuration record associated with a caller-supplied object.

A zero second argument disables the entire operation. Every nonzero second argument selects the same construction path and is ultimately written, without Boolean normalization, into the configuration record.

Static evidence establishes the field accesses, numeric widths, call sequence, initialization values, and connection order described below. The application-level meanings of the resource, auxiliary objects, and numeric pairs are not established. A geometric interpretation is plausible but is not part of the required behavior.

## Interface

### Arguments and result

| Input or output | Contract |
|---|---|
| First argument | A 32-bit value used as the caller-object address on the enabled path |
| Second argument | A 32-bit integer: zero disables construction; any nonzero value enables it |
| Return | No explicit result or status value |

For normal completion on the enabled path, all accessed addresses must support the reads and writes described here. The function checks neither the caller-object address nor addresses returned by dependencies.

The caller object has accessed fields extending through byte offset `0x77`. This does not imply that every preceding byte is read or written.

### Caller-object fields

Each listed access is 32 bits wide.

| Offset | Observed use |
|---|---|
| `0x20` | First endpoint used in extent B |
| `0x24` | First endpoint used in extent A |
| `0x28` | Second endpoint used in extent B |
| `0x2C` | Second endpoint used in extent A |
| `0x68` | Published configuration address |
| `0x6C` | Recorded resource value |
| `0x70` | Recorded second auxiliary-object address |
| `0x74` | Recorded first auxiliary-object address |

**Extent A** denotes the 32-bit subtraction of the value at `0x24` from the value at `0x2C`. **Extent B** denotes the corresponding subtraction of `0x20` from `0x28`.

These names describe arithmetic, not validated dimensions. The function recomputes extents at multiple points; they are not immutable values captured at entry.

### Auxiliary-object fields

Each selected auxiliary object must support the following 32-bit accesses:

| Offset | Observed use |
|---|---|
| `0x04` | Read for cleanup and binding; conditionally cleared |
| `0x0C` | Read for binding |
| `0x10` | Initially cleared; subsequently tested and conditionally cleared |

The directly accessed range extends through byte offset `0x13`. Dependency accesses may impose additional requirements that are not available here.

### Configuration layout

The function requests 40 bytes for this record.

| Offset | Initial direct write | Later connection write |
|---|---|---|
| `0x00` | 32-bit zero | First auxiliary-object address |
| `0x04` | 32-bit zero | None |
| `0x08` | None | Resource value |
| `0x0C` | 32-bit zero | Second auxiliary-object address |
| `0x10` | 32-bit zero | Original second argument |
| `0x14` | 32-bit `0x00010000` | None |
| `0x18` | 32-bit `0x00010000` | None |
| `0x1C` | 32-bit `0x00010000` | None |
| `0x20` | 32-bit `0x00010000` | None |
| `0x24` | One-byte zero | None |
| `0x25`–`0x27` | None | None |

No allocator-zeroing guarantee is supplied. In particular, the resource field is not initialized before publication, and the last three bytes receive no direct write from this function.

## Behavior

### Disabled invocation

When the second argument is zero, the function returns without accessing the caller object or the described globals, invoking dependencies, allocating storage, or writing memory.

The first argument therefore need not identify accessible storage on this path.

### Resource creation and publication

On the enabled path, F-a29c8be2c8c1 receives:

1. A 32-bit value read through a global location.
2. Zero.
3. Zero.
4. Extent A.
5. Extent B.

The returned 32-bit value is immediately written to caller-object offset `0x6C`. This happens before temporary-pair preparation, auxiliary-object creation, or configuration allocation. The result is not checked for validity.

The supplied representation does not establish the relative order of independent global and endpoint reads within this call’s argument preparation.

### Temporary-pair preparation

Eight temporary regions, each eight bytes long, are passed separately to F-6487171f4e1c. Each call supplies a destination and two 32-bit numeric arguments. The destinations form two ordered groups of four regions.

For the first group, the numeric arguments are:

| Pair | First component supplied | Second component supplied |
|---|---|---|
| First | Result of F-870c4fcf7295 for extent A and `0xFFFFFFFE`, shifted left 16 bits | Signed half of extent B, shifted left 16 bits |
| Second | Signed half of extent A, shifted left 16 bits | Signed half of extent B, shifted left 16 bits |
| Third | Result of F-870c4fcf7295 for extent A and `0xFFFFFFFE`, shifted left 16 bits | Result of F-870c4fcf7295 for extent B and `0xFFFFFFFE`, shifted left 16 bits |
| Fourth | Signed half of extent A, shifted left 16 bits | Result of F-870c4fcf7295 for extent B and `0xFFFFFFFE`, shifted left 16 bits |

**F-870c4fcf7295 is called exactly four times on a normally completing enabled path, in extent order A, B, A, B.** The first pair accounts for the first A call. The third pair accounts for the next two calls, evaluated B before A. The fourth pair accounts for the final B call. The second pair does not call this dependency.

The second group receives these component pairs, in order:

1. Zero and `0x00010000`.
2. `0x00010000` and `0x00010000`.
3. Zero and zero.
4. `0x00010000` and zero.

All four first-group preparation calls precede all four second-group preparation calls. All eight precede auxiliary-object creation.

The first group’s base address is later supplied to the first population call; the second group’s base address is supplied to the second. Each population call also receives four and two.

The layout and arguments support interpreting each group as four two-component entries. The actual writes performed by F-6487171f4e1c are not established by this function’s direct operations.

### Arithmetic and mutation-sensitive reads

Subtractions, helper results, division operands, and shift expressions are represented as 32-bit quantities. There is no established widening to arbitrary-precision arithmetic.

Division by two is signed integer division with truncation toward zero. For representable inputs, five halves to two, and negative five halves to negative two. Shifting follows division; this is not division of an already shifted fixed-point quantity.

Each left shift is by 16 bits. A contract must not replace it with an unrestricted mathematical product that preserves additional high bits. Exact target behavior for overflowing subtraction, negative shifted values, or out-of-range shifted results remains unresolved.

The helper’s second argument has bit pattern `0xFFFFFFFE`, representing negative two under signed 32-bit interpretation. The operation performed by F-870c4fcf7295 is unknown.

Read timing matters:

- After resource publication, the two endpoint values for the first pair’s B component are read before the first A-helper call. Those retained endpoint values supply that pair’s half-B calculation.
- The second pair uses fresh endpoint reads after the first pair-preparation call returns.
- For the third pair, the B-helper result is obtained before the A-helper call. The B result is retained while the A call executes.
- For the fourth pair, the B-helper call occurs before the endpoint reads used to calculate half of A.

Accordingly, a dependency that changes caller fields can affect later components without changing already retained operands or results. Relative ordering of otherwise independent reads inside a single argument list is not fully established.

### Auxiliary-object creation

After preparing both temporary groups, the function:

1. Calls F-8157a180d901 with 20.
2. Calls F-60a658d6910b without explicit arguments.
3. Writes a 32-bit zero at offset `0x10` of the returned address.
4. Stores that address at caller-object offset `0x74`.

It repeats the same sequence, storing the second returned address at caller-object offset `0x70`.

The supplied representation does not explicitly pass the size-20 call’s return value to F-60a658d6910b. Their allocator/constructor relationship is therefore unresolved; only the call sequence and subsequent address use are established.

Neither returned auxiliary address is checked before dereference.

### Configuration initialization and publication

F-8157a180d901 is next called with 40. Its return value is directly used as the configuration address.

Initialization proceeds in this order:

- 32-bit zero writes at `0x00`, `0x04`, `0x0C`, and `0x10`.
- 32-bit `0x00010000` writes at `0x14`, `0x18`, `0x1C`, and `0x20`.
- A one-byte zero write at `0x24`.

The configuration address is then written to caller-object offset `0x68`.

This publication precedes auxiliary population and binding. At publication, the function has not written configuration offset `0x08`, and the connection fields still contain their initial zeros unless dependency effects or aliasing have changed them.

### First auxiliary-object processing

The function reloads the first auxiliary-object address from caller-object offset `0x74`. It retains that selected address through this object’s cleanup, population, and binding.

If the selected object’s 32-bit `0x10` field is nonzero:

1. Its `0x04` field is read and passed to F-58ba61416cc9.
2. After the call returns, `0x04` is cleared.
3. Then `0x10` is cleared.

The earlier zero initialization does not prove this branch unreachable: intervening dependencies may have changed the object or the caller’s recorded address. Cleanup does not reselect the object after the dependency returns.

A first global byte flag is then tested. If zero, it is set to one before F-0fd46b9a392f is called with the first registration token. If already nonzero, that registration call is skipped.

F-ab404c034f3b next receives the selected object, the first temporary-group address, four, and two.

After population returns, a shared 32-bit binding token is captured. F-f7dc68efe170 receives:

1. The selected object’s current `0x0C` field.
2. The shared binding token.
3. Zero.
4. The selected object’s current `0x04` field.

The token capture and first binding argument preparation have no intervening dependency call. The supplied representation does not justify assumptions about concurrent changes during these reads.

### Second auxiliary-object processing

After the first binding returns, the second auxiliary-object address is reloaded from caller-object offset `0x70`.

Its `0x10` field controls the same cleanup sequence: pass the current `0x04` value to F-58ba61416cc9, then clear `0x04`, then clear `0x10`.

The second registration token is captured before testing the second global byte flag. If that flag is zero, the function sets it to one and invokes F-0fd46b9a392f with the captured token.

F-ab404c034f3b then receives the selected second object, the second temporary-group address, four, and two.

F-1417de9253c4 subsequently receives the object’s current `0x0C` field, the shared binding token retained from the first stage, zero, and the object’s current `0x04` field. The shared token is not recaptured for this binding.

### Final connections

After the second binding returns, four 32-bit writes occur in order:

1. Configuration `0x08` receives caller-object `0x6C`.
2. Configuration `0x00` receives caller-object `0x74`.
3. Configuration `0x0C` receives caller-object `0x70`.
4. Configuration `0x10` receives the original second argument.

The destination configuration address is obtained through caller-object `0x68` for each assignment. The other source fields are also read at this final stage, rather than supplied from the earlier creation results.

Thus, dependencies that redirect caller fields can change the final destinations or connections. The writes are not unconditionally guaranteed to target the originally allocated configuration or reference the originally created auxiliary objects.

## Side effects

The enabled path directly overwrites caller-object offsets `0x68`, `0x6C`, `0x70`, and `0x74`. It does not first release their previous contents.

It also initializes auxiliary state fields, initializes and connects configuration storage, and may change two global byte flags from zero to one. Dependency side effects are additional and unspecified.

No direct release of the created resource, auxiliary objects, or configuration appears. Passing an auxiliary field to F-58ba61416cc9 does not establish whether that dependency releases, transfers, or merely observes the value.

Temporary storage is invocation-local. Dependencies receive its addresses, but copying versus retention is unresolved. Safe use of retained temporary addresses after return is not established.

There is no explicit rollback, error return, allocation-failure branch, or recovery path. If execution fails partway through, earlier publications and flag writes have already occurred. In particular, a registration flag is set before the registration call returns successfully.

Repeated invocation is not established as replacement-safe or idempotent. Aliasing among supplied, returned, or dependency-modified addresses is not checked. Thread safety and reentrancy are not established.

## Dependencies

| Opaque ID | Established interaction |
|---|---|
| F-a29c8be2c8c1 | Receives a global value, two zeros, A, and B; returns the recorded resource value |
| F-870c4fcf7295 | Four calls in extent order A, B, A, B; each also receives `0xFFFFFFFE` |
| F-6487171f4e1c | Eight calls, each receiving an eight-byte temporary destination and two numeric arguments |
| F-8157a180d901 | Receives sizes 20, 20, and 40; the final result is directly used as configuration storage |
| F-60a658d6910b | Two calls without explicit arguments; each result is used as an auxiliary-object address |
| F-58ba61416cc9 | Receives `0x04` when the selected object’s `0x10` field is nonzero |
| F-0fd46b9a392f | Receives registration tokens under separate byte-flag guards |
| F-ab404c034f3b | Populates each selected auxiliary object using its corresponding temporary group, four, and two |
| F-f7dc68efe170 | Receives the first object’s binding arguments |
| F-1417de9253c4 | Receives the second object’s binding arguments |

No dependency is assumed to preserve globals, caller fields, auxiliary fields, or unrelated reachable memory. Ownership and failure semantics require independent contracts.

## Acceptance examples

These examples are conditional specifications, not executed tests.

1. **Disabled call.** With a zero second argument, there are no described memory accesses or dependency calls, even if the first argument is not a valid object address.

2. **Helper count and order.** On an enabled invocation that reaches completion, exactly four F-870c4fcf7295 calls occur. They process A for the first pair, B then A for the third pair, and B for the fourth pair. No fifth helper call occurs.

3. **Stable positive extents.** If endpoint fields remain unchanged with A equal to ten and B equal to six, the resource dependency receives ten and six. The second pair-preparation call receives numeric components `0x00050000` and `0x00030000`. Helper-derived components remain conditional on the helper’s unknown operation.

4. **Odd positive extent.** With stable B equal to five, a half-B component is two before shifting, yielding `0x00020000`, not `0x00028000`.

5. **First-pair read timing.** If B is six when its endpoints are retained, and the first A-helper call changes those endpoints to make B ten, the first pair still receives a half-B component of `0x00030000`. A later freshly read half-B component can instead be `0x00050000`, provided no further mutation intervenes.

6. **Non-Boolean enabled value.** With second argument negative three and valid normal completion, the final configuration write at `0x10` stores its full 32-bit representation, `0xFFFFFFFD`, rather than one.

7. **Registration already marked.** If both byte flags are nonzero when tested, neither guarded registration call occurs. Population and binding still proceed.

8. **Binding-token mutation.** If the first binding dependency changes the global shared token after its capture, the second binding still receives the retained token. Changes before capture can affect that retained value.

9. **Conditional cleanup.** If a selected object’s `0x10` field is nonzero, its current `0x04` value is passed to F-58ba61416cc9 before the function clears `0x04` and then `0x10`.

10. **Stable completion.** If dependencies provide valid storage and preserve relevant fields, final configuration offsets `0x00`, `0x08`, and `0x0C` reference the first auxiliary object, resource value, and second auxiliary object. Bytes `0x25`–`0x27` remain unspecified.

## Uncertainties

The following remain **documented_only**:

- Application semantics of the resource, pair groups, registration tokens, and binding token.
- The operation and failure behavior of F-870c4fcf7295.
- Exact dependency writes, temporary-data copying, and reference-retention requirements.
- Any hidden relationship between size-20 allocation calls and auxiliary-object construction.
- Allocation failure, ownership transfer, release obligations, and dependency failure behavior.
- Target-level overflowing arithmetic and negative or out-of-range shift behavior.
- Dependency mutation of fields and globals, address aliasing, concurrency, and reentrancy.
- Relative ordering of independent reads not explicitly sequenced by the supplied representation.

Interpreting `0x00010000` as fixed-point one, the first group as geometric coordinates, or the second group as normalized coordinates is inferred intent only.

## Evidence

This contract uses the supplied static representation as evidence for explicit branches, widths, offsets, constants, temporary-region sizes, dependency calls, retained values, and write order.

The four numeric-helper calls are individually accounted for: one A call for the first pair, B then A calls for the third pair, and one B call for the fourth pair. The resulting extent sequence is A, B, A, B.

Dependency internals, application semantics, and target-level behavior beyond the supplied representation are not established. No runtime tests have been performed.
