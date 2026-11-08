<!-- Trusted provenance: F-f076cfabe3e2; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-f076cfabe3e2

## Purpose

F-f076cfabe3e2 selects between two execution paths using a 32-bit explicit input and, when that input is nonzero, a condition in shared state.

The larger path invokes an ordered sequence of dependencies, clears three context fields, passes a separately read context value to each of two dependencies, and supplies a partially initialized temporary record to another dependency.

The smaller path invokes a dependency on an embedded context address. After that dependency returns, it loads a shared receiver pointer. A nonnull receiver leads to indirect dispatch whose second and third arguments are values read from two distinct shared-storage locations.

**Established static evidence:** the supplied representation establishes the conditional control flow, explicit call operands, context access widths and offsets, and temporary-record assignments described below.

**Inferred intent:** the larger path resembles a coordinated state transition followed by notification. The smaller path resembles an alternative handler. Neither interpretation establishes application semantics or dependency behavior.

**Unknown:** the operation’s application-level purpose, complete calling convention, dependency contracts, and validity rules for the referenced objects.

## Interface

### Explicit input and result

The explicit input is represented as a 32-bit integer.

- Zero selects the smaller path without evaluating the shared-state eligibility condition.
- Any nonzero value causes that condition to be evaluated.
- No magnitude-dependent distinction is visible. Positive and negative nonzero values have the same selection behavior.

The function has no declared return value. Normal completion supplies no specified result or status code.

### Implicit context

The function uses a context address supplied through incoming machine state outside the recovered explicit parameter list. This contract calls it **C**. Its origin is unresolved; the explicit integer input is not shown to provide it.

| Offset from C | Access width | Visible use |
|---|---:|---|
| `0x30` | 32 bits | Read separately for two larger-path calls |
| `0x4D3` | 8 bits | Written with zero on the larger path |
| `0x728` | 32 bits | Written with zero on the larger path |
| `0x72C` | 32 bits | Written with zero on the larger path |
| `0x82C` | Address computation only | Passed to the first smaller-path dependency |

Passing C plus `0x82C` does not establish the extent, type, or layout of the object beginning there.

### Shared storage

The following labels describe storage roles, not application-level types:

| Label | Role |
|---|---|
| **G-A** | Shared base address with a represented 32-bit pointer field at offset `0x10`, used for path selection |
| **G-B** | Shared base address with a represented 32-bit pointer field at offset `0x14`; the referenced object’s 32-bit field at `0x30` controls an optional call |
| **G-D** | Shared storage from which the smaller-path receiver pointer is loaded |
| **G-E** | Distinct shared storage whose value supplies the second indirect-dispatch argument |
| **G-F** | Distinct shared storage whose value supplies the third indirect-dispatch argument |

G-E and G-F supply dynamically read values, not established constants. Their values, access widths, and application meanings are unresolved.

The working layout model uses 32-bit integers and addresses, with explicitly identified 8-bit accesses where shown. This does not establish a complete ABI or determine the widths of otherwise untyped operands. Endianness is not established.

### Preconditions

Safe normal execution requires all addresses actually accessed on the selected path to be valid for their respective accesses. No general address validation is visible.

In particular:

- C must support the selected path’s accesses.
- Shared bases and dereferenced intermediate pointers must be valid.
- A nonnull smaller-path receiver must have a readable first pointer field, and the resulting address must support lookup of a callable entry at byte offset `0x58`.
- G-E and G-F must support the accesses supplying the dispatch arguments when dispatch occurs.
- Each dependency must satisfy its own unknown preconditions.

These are requirements for valid execution, not checks performed by this function. Address calculations use constant byte offsets in the represented 32-bit model. No bounds or overflow checks are visible, and wrapped addresses are not thereby valid objects.

## Behavior

### Path selection

A zero explicit input selects the smaller path immediately. The eligibility condition under G-A is not evaluated.

For a nonzero input, the function first reads the pointer field at G-A plus `0x10`.

- A null value selects the larger path without reading an eligibility byte.
- A nonnull value causes another read of that pointer field, followed by a one-byte read at offset `0x20` from the object identified by the second pointer value.
- A nonzero byte selects the larger path.
- A zero byte selects the smaller path.

The supplied representation contains separate pointer-field reads when the first value is nonnull. It does not establish a stable snapshot. If the second value differs, that second value determines the byte-access address. The initial null check therefore does not protect against an invalid or null second value.

These are represented accesses; exact machine-level load realization and concurrency guarantees are not independently established.

### Larger path: initial calls and optional operation

On normal continuation, the first calls are:

1. F-753f887b468d.
2. F-5e7d48ce1b92.

No explicit arguments are shown.

Only after these calls does the function read the pointer field at G-B plus `0x14`, then the 32-bit field at offset `0x30` in the referenced object. A nonzero field causes a call to **D-01**; zero skips it. No explicit arguments are shown for D-01.

The intermediate pointer is not visibly checked for null. Dependencies preceding this read may affect the shared state; the function does not visibly capture the controlling field before them.

Execution then invokes, in order:

1. F-5f6123f0d287.
2. F-4c05775b039e.
3. F-ebab728f78fa.
4. F-f41bc2c113a5.
5. F-6ac70711bbb5.
6. F-d4185931348b.

No explicit arguments are shown for these calls.

### Larger path: context writes and continuation

After those calls, three direct stores occur in the represented order:

1. An 8-bit zero at C plus `0x4D3`.
2. A 32-bit zero at C plus `0x728`.
3. A 32-bit zero at C plus `0x72C`.

The byte store does not clear adjacent bytes. The two word stores do not establish a reset of any larger structure.

The next calls are:

1. F-a686dc50fef4 with explicit argument `1`.
2. F-88d8cdcbf029.
3. F-6aae495eebbc.
4. F-87be4aa684d1.

The latter three calls show no explicit arguments. The 32-bit result from F-87be4aa684d1 controls one additional call:

- Nonzero invokes F-4f1654b28d1d, with no explicit arguments shown.
- Zero skips that call.

Both cases then continue with:

1. F-d260cfbdb3c8.
2. F-acc006b5b6e3.
3. F-bfade2068003 with explicit arguments `3` and `0`, in that order.

The first two show no explicit arguments.

The function next reads the 32-bit value at C plus `0x30` and passes it to F-e6177ac586d2. After that dependency returns, it reads the same location again and passes the newly read value to F-a80bb5383b07.

The two arguments are not guaranteed to be equal. In particular, F-e6177ac586d2 may change the field before the second read; no preservation guarantee is supplied.

### Larger path: temporary record

After the two context-value calls, selected locations in automatic storage are initialized. Let **R** be the address subsequently passed to D-03.

| Offset from R | Width | Assigned value |
|---|---:|---:|
| `0x00` | 32 bits | `0` |
| `0x04` | 32 bits | `0` |
| `0x08` | 32 bits | `0` |
| `0x0C` | 32 bits | `0` |
| `0x10` | 8 bits | `2` |
| `0x11`–`0x13` | 3 bytes | No initialization shown |
| `0x14` | 32 bits | `3` |
| `0x18` | 32 bits | `1001`, equivalent to `0x3E9` |
| `0x1C`–`0x1F` | 4 bytes | No initialization shown |
| `0x20` | 32 bits | `0` |

The shown locations span 36 bytes. They do not establish a completely initialized 36-byte object or its complete extent.

The represented assignment order is the four zero words at offsets `0x00` through `0x0C`, the words at `0x14`, `0x18`, and `0x20`, then the byte at `0x10`. All precede the following calls. Exact physical ordering of ordinary stack writes is not independently established.

The final calls are:

1. **D-02** with explicit argument `9`.
2. **D-03** with explicit arguments `1`, R, `5`, and `6`, in that order.
3. **D-04** with explicit argument `9`.

The function returns after D-04 completes normally; it does not proceed into the smaller path.

The record’s type, field meanings, and extent consumed by D-03 are unknown. Its state at D-03 entry is not guaranteed solely by the initial assignments: D-02 intervenes, and no general preservation contract for that dependency is supplied.

### Smaller path

The smaller path first calls F-ec4cfa552cce with C plus `0x82C`.

After that call returns, the function loads the receiver pointer from G-D.

- If the receiver is null, the function returns. The dispatch expression is not evaluated, so it supplies no reads of G-E or G-F and performs no indirect-target lookup.
- If the receiver is nonnull, the function obtains an indirect target using the receiver’s first pointer field and an entry at byte offset `0x58` from that field’s value.

The indirect dispatch receives three explicit arguments:

1. The receiver loaded from G-D.
2. A value read from G-E.
3. A value read from G-F.

The G-E and G-F reads occur after F-ec4cfa552cce returns and only in the nonnull-receiver case. That dependency may change either shared value, as well as the receiver storage.

The supplied call expression does **not** establish a relative evaluation order between the G-E read, the G-F read, and the indirect-target lookup. Argument position establishes which shared value supplies which argument; it does not establish read order. No coherent snapshot across these accesses is guaranteed.

Neither the receiver’s first field nor the callable entry is visibly checked for validity. No dispatch result is visibly used.

### Errors and interruption

No explicit error result, retry, rollback, or recovery handler is shown.

Only the result of F-87be4aa684d1 is visibly tested. Unused results do not establish that other dependencies cannot fail.

The sequences above are conditional on normal continuation. A dependency that does not return normally may prevent all later calls and stores. Invalid memory accesses and invalid indirect targets have no specified recovery behavior in the supplied evidence.

## Side effects

The larger path directly writes the three context fields and selected temporary-record locations. Its dependencies may produce additional effects.

The smaller path has no explicit persistent store in the supplied body. It reads shared storage and, for a nonnull receiver, reads the indirect target and dispatch operands before invoking the target.

Dependency side effects remain opaque. No dependency is established here as an allocator, destructor, synchronization operation, or resource-release operation.

Ownership is unresolved:

- Passing the value from C plus `0x30` establishes neither transfer nor borrowing rules.
- Passing C plus `0x82C` establishes no copy, ownership transfer, or release.
- The receiver has no visible acquisition or release.
- R addresses automatic storage. Passing it does not prove that D-03 copies, modifies, or retains the referenced data. Its storage lifetime does not extend beyond this invocation.

The three cleared context fields are not guaranteed to remain zero at return because dependencies execute after the stores. Likewise, shared values are not assumed to remain unchanged across calls.

## Dependencies

All mapped direct dependencies are identified by opaque IDs in the behavior sections. Their visible roles are limited to call order, explicit operands, and the conditional result use described there.

The remaining dependencies have local opaque IDs:

| ID | Visible role |
|---|---|
| D-01 | Optional larger-path call controlled by the field reached through G-B |
| D-02 | First final call; receives `9` |
| D-03 | Receives `1`, R, `5`, `6` |
| D-04 | Last larger-path call; receives `9` |
| D-05 | Smaller-path indirect target, resolved through the nonnull receiver |

D-01 through D-04 are unmapped in the supplied dependency map. D-05 is unresolved indirect dispatch, not a fixed known implementation.

“No explicit arguments shown” does not prove absence of hidden inputs or reliance on incoming machine state. No dependency is assumed to preserve globals, preserve context memory, or own or release passed storage.

## Acceptance examples

These are conditional criteria from static evidence, not executed tests. Unless otherwise stated, accessed memory is valid and dependencies return normally.

1. **Zero input and null receiver.** The function calls F-ec4cfa552cce with C plus `0x82C`, then loads G-D. If that receiver is null, it returns without evaluating the eligibility condition, reading G-E or G-F for dispatch, or looking up an indirect target.

2. **Nonzero input and initially null eligibility pointer.** A null first read at G-A plus `0x10` selects the larger path without reading the eligibility byte.

3. **Zero eligibility byte.** With a nonzero input, a nonnull initial eligibility pointer, and a valid second pointer identifying a zero byte at offset `0x20`, the smaller path is selected. The larger path’s three direct context stores do not occur.

4. **Nonzero eligibility byte.** Any nonzero byte at the selected eligibility location chooses the larger path. Equality to `1` is not required.

5. **Independent optional calls.** A zero 32-bit controlling field reached through G-B skips D-01; nonzero invokes it. Independently, a zero result from F-87be4aa684d1 skips F-4f1654b28d1d; nonzero invokes it.

6. **Context value changes between calls.** If the first read at C plus `0x30` yields A and F-e6177ac586d2 changes the field to B, with no further change before the second read, F-a80bb5383b07 receives B.

7. **Receiver changed by the first smaller-path call.** If F-ec4cfa552cce changes G-D to another valid receiver, dispatch uses the subsequently loaded receiver. If it makes G-D null, no dispatch occurs.

8. **Dispatch operands changed by the first smaller-path call.** If F-ec4cfa552cce sets G-E to A and G-F to B, the subsequent receiver is nonnull, and those values remain stable through argument evaluation, dispatch receives that receiver, A, and B. The operands are not values captured before the dependency call.

9. **Changing dispatch storage.** If G-E or G-F changes during dispatch-expression evaluation, acceptance must not require a particular relative read order or an atomic pair. The evidence establishes operand sources and null gating, not snapshot semantics.

10. **Temporary-record gaps.** Acceptance must not require bytes `0x11`–`0x13` or `0x1C`–`0x1F` to be zero. It must also not assume D-03 leaves the record unchanged.

## Uncertainties

The following remain **documented_only**:

- How C is supplied and whether the recovered explicit signature is complete.
- Application meanings and complete object layouts.
- Values, access widths, and meanings of the shared operands from G-E and G-F.
- Relative evaluation order among those operand reads and the indirect-target lookup.
- ABI details, alignment guarantees, endianness, and exact indirect-transfer mechanics.
- Dependency hidden inputs, side effects, preservation rules, failure modes, and ownership contracts.
- D-03’s read extent, treatment of uninitialized gaps, modifications, copying, or address retention.
- Whether shared storage can change concurrently and whether dependencies provide synchronization.
- Exact machine-level correspondence of the represented memory accesses and ordering.

These unknowns do not justify treating shared operands as constants, assuming callee preservation, or supplying missing ownership behavior.

## Evidence

This contract uses only the supplied static function representation and dependency map. Prior prose, comments, recovered names, and reviewer assertions are not independent behavioral proof.

The dispatch expression supplies its second and third arguments from two distinct shared-storage values. Its placement after the first smaller-path call and the null-receiver return establishes when those argument reads can occur. It does not establish their relative evaluation order or their order relative to target lookup.

The representation also supports the stated branches, ordered call boundaries, context access widths and offsets, and partial temporary-record initialization. It does not establish dependency implementations or application semantics.

No runtime execution, smoke checks, or tests have been performed. All acceptance examples are conditional static criteria, not observed results.
