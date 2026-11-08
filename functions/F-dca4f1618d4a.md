<!-- Trusted provenance: F-dca4f1618d4a; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-dca4f1618d4a

## Purpose

F-dca4f1618d4a obtains a state-handle representation, obtains an unsigned 32-bit baseline from F-be4e5c73e101, computes a replacement using a signed adjustment, submits that replacement to F-829bd0265134, and returns the result of a terminal indirect operation on the receiver.

The supplied representation establishes dependency invocations, not their internal storage behavior. In particular, it does not establish that F-be4e5c73e101 reads a stored quantity or that F-829bd0265134 writes the submitted replacement unchanged.

Two mutually exclusive paths are represented: nonpositive adjustment and positive adjustment. Each ends with an indirect operation selected through the receiver’s operation table. The paths supply arguments from different pairs of global data locations. Those global contents are read during terminal dispatch, after F-829bd0265134 returns; they are not established constants or earlier snapshots.

**Inferred intent:** the computation resembles adjustment of a quantity toward boundaries of 0 and 100, with an additional operation for changes strictly inside a boundary. This is not a general range guarantee. Positive adjustments greater than 100 expose unsigned wraparound in the comparison threshold.

A trailing region describes receiver-local updates and temporary-storage operations. The displayed branch conditions and returns provide no ordinary entry into that region.

## Interface

### Inputs and output

| Item | Contract |
|---|---|
| Receiver | Address of an object whose first word supplies an operation-table address |
| Adjustment | Signed 32-bit integer; zero selects the nonpositive path |
| Return value | Unsigned 32-bit result of the terminal indirect operation, without visible normalization |

The receiver and original adjustment are also passed to F-2dfa0238c98f when the selected path’s strict comparison succeeds.

The function does not return the computed replacement or an indication that F-2dfa0238c98f was called.

### Relevant layouts and widths

| Location | Established use |
|---|---|
| Provider result, byte offset 0x28 | Read a four-byte state-handle representation |
| Receiver, byte offset 0 | Read the operation-table address |
| Operation table, byte offset 0x58 | Select the terminal indirect operation |

The supplied representation uses signed and unsigned 32-bit arithmetic and is consistent with a 32-bit pointer layout. Exact machine calling conventions and the underlying type of the state handle are not independently established.

Four distinct global data locations supply the terminal arguments. Stable opaque labels are assigned here by branch and argument position:

| Global label | Branch | Terminal argument position |
|---|---|---|
| G-01 | Nonpositive | Second, immediately after the receiver |
| G-02 | Nonpositive | Third |
| G-03 | Positive | Second, immediately after the receiver |
| G-04 | Positive | Third |

These labels identify locations, not fixed values. The source does not supply their declarations, widths, application meanings, or contents. The four locations are distinct; their contents need not differ.

### Preconditions for normal completion

- F-d189d16d5a68 returns an address permitting a four-byte read at offset 0x28.
- The captured handle is acceptable to the selected dependencies.
- The receiver, its table, and the selected operation remain valid when terminal dispatch occurs.
- The selected global locations are readable when terminal arguments are evaluated.
- Every invoked dependency returns normally.

No visible null checks, handle validation, table validation, or recovery paths enforce these conditions. No particular outcome is specified here for their violation.

## Behavior

### Common acquisition sequence

For either reachable branch, the observable dependency and memory-access sequence begins as follows:

1. Invoke F-d189d16d5a68 without explicit arguments.
2. Read the four-byte field at offset 0x28 of its result and retain that handle representation.
3. Invoke F-be4e5c73e101 with the captured handle.
4. Retain its unsigned 32-bit result as the baseline.

Each invocation selects only one branch and therefore performs this sequence once.

The baseline is a dependency result, not an independently verified snapshot of stored state. Nevertheless, the function retains that result for its own comparison and arithmetic. It does not reacquire the handle or obtain another baseline after the optional call to F-2dfa0238c98f.

### Nonpositive adjustment

An adjustment less than or equal to zero selects this path.

The function compares the adjustment’s negated value, converted to unsigned 32-bit form, with the baseline. Except for the most-negative signed input discussed below, this is a comparison of the adjustment’s magnitude against the unsigned baseline.

If the magnitude is strictly less than the baseline:

- Invoke F-2dfa0238c98f with the receiver and the original signed adjustment.
- Compute the replacement by adding the original adjustment to the retained baseline using the represented unsigned 32-bit arithmetic.

Otherwise:

- Do not invoke F-2dfa0238c98f.
- Select replacement zero.

The function then invokes F-829bd0265134 with the captured handle and the replacement.

After that invocation returns, terminal dispatch selects the operation at receiver-table offset 0x58. Its arguments are:

1. The receiver.
2. The contents read from G-01.
3. The contents read from G-02.

The indirect operation’s unsigned 32-bit result becomes the function’s return value.

Consequences include:

- Equality between negative-adjustment magnitude and baseline skips F-2dfa0238c98f and submits zero.
- A larger negative magnitude also skips that call and submits zero.
- Zero adjustment with nonzero baseline invokes F-2dfa0238c98f with zero, then submits the unchanged baseline.
- Zero adjustment with zero baseline skips F-2dfa0238c98f but still invokes F-829bd0265134 and terminal dispatch.

Zero adjustment is therefore not an observable no-op.

### Positive adjustment

A strictly positive adjustment selects this path.

The comparison threshold is unsigned 32-bit subtraction of the adjustment from 100. If the unsigned baseline is strictly below that threshold:

- Invoke F-2dfa0238c98f with the receiver and the original positive adjustment.
- Compute the replacement by adding that adjustment to the retained baseline using unsigned 32-bit arithmetic.

Otherwise:

- Do not invoke F-2dfa0238c98f.
- Select replacement 100.

The function then invokes F-829bd0265134 with the captured handle and the replacement.

After that invocation returns, terminal dispatch selects the operation at receiver-table offset 0x58. Its arguments are:

1. The receiver.
2. The contents read from G-03.
3. The contents read from G-04.

The indirect operation’s unsigned 32-bit result becomes the function’s return value.

For adjustments from 1 through 100, the arithmetic resembles saturation at 100. A sum strictly below 100 takes the optional-operation path; reaching exactly 100 skips that operation and submits 100. This interpretation does not extend to adjustments greater than 100.

### Terminal reads and ordering limits

On either path, F-829bd0265134 returns before evaluation of the terminal global operands and receiver-table lookup.

The supplied representation does **not** establish an evaluation order among:

- Reading the selected first global operand.
- Reading the selected second global operand.
- Reading the receiver’s table address and selecting its operation.

Argument position is established; temporal order among those reads is not.

No earlier capture of these global contents or the terminal table address is visible. Any preceding dependency may change them. Consequently:

- The selected global contents need not equal their contents at function entry.
- The optional operation or F-829bd0265134 may affect the eventual terminal arguments.
- A changed receiver-table address may affect the terminal target, provided the receiver remains valid.

These observations do not imply that any dependency actually performs such mutations.

### Integer boundaries

The baseline, positive-path threshold, and addition expressions use unsigned 32-bit arithmetic.

For positive adjustments greater than 100, threshold subtraction wraps modulo \(2^{32}\). Adjustment 101 therefore produces threshold 4,294,967,295 rather than a negative threshold. Baseline 20 is below that threshold, so the function invokes F-2dfa0238c98f and computes replacement 121.

Addition can also wrap modulo \(2^{32}\). With baseline 4,294,967,294 and adjustment 101, the strict comparison succeeds and the arithmetic result is 99.

The computed unsigned result is carried through a signed 32-bit slot in the supplied representation before being passed onward. For results outside the signed range, the exact signed conversion and dependency argument interpretation are not independently established. The contract identifies the unsigned arithmetic result; it does not invent a signed numeric interpretation for the recipient.

For adjustment −2,147,483,648, signed negation is outside the signed 32-bit range. The supplied representation does not establish the actual instruction behavior at that boundary. Under a two’s-complement wrapping interpretation, the unsigned comparison magnitude would be 2,147,483,648. That interpretation is conditional, not an unconditional language-level guarantee.

### Trailing region

Every signed adjustment satisfies either the nonpositive or positive condition, and both represented branches return. No ordinary input therefore reaches the trailing region under the displayed control flow.

That region references:

- Signed receiver fields at byte offsets 0x78, 0x7C, and 0x80.
- An associated-object reference at receiver offset 0x1C.
- An indirect operation at associated-object table offset 0xB8.
- An indirect operation at offset 0xE4 on another dependency result.
- Temporary storage of 16 and 24 bytes.
- Additional direct dependencies, another receiver dispatch, and a Boolean-shaped return.

These observations do not establish reachable receiver writes, temporary-object lifetimes, cleanup, or Boolean return behavior for this function. The four global labels above describe only the two reachable terminal argument pairs, not operands appearing solely in this trailing region.

## Side effects

### Established effects and sequencing

Assuming normal returns, each reachable invocation performs:

- One invocation of F-d189d16d5a68.
- One four-byte handle-field read.
- One invocation of F-be4e5c73e101.
- Zero or one invocation of F-2dfa0238c98f.
- Exactly one invocation of F-829bd0265134.
- Terminal receiver-table lookup and reads of the selected global pair.
- Exactly one terminal indirect invocation.

F-829bd0265134 is invoked even when the replacement equals the baseline. Its invocation precedes terminal dispatch.

The reachable portion contains no direct receiver-field write and no direct write to the four terminal globals. Dependencies may mutate the receiver, globals, or other state; their internals are not supplied.

### Retained values

The captured handle and baseline survive the optional call as inputs to the function’s subsequent computation. If F-2dfa0238c98f changes externally held state, the function still uses its retained baseline and captured handle.

This differs from terminal arguments: those global contents are read later, after F-829bd0265134 returns. No preservation across preceding calls is assumed.

### Ownership and failures

No explicit allocation, release, retain, or ownership-transfer operation appears in the reachable portion. This absence does not establish that dependencies preserve ownership or allocate nothing.

No result from F-2dfa0238c98f or F-829bd0265134 is visibly checked. There is no represented rollback or recovery. If terminal dispatch fails to complete, the earlier dependency invocations have already occurred.

## Dependencies

| Dependency | Observed role | Unestablished properties |
|---|---|---|
| F-d189d16d5a68 | Supplies the address containing the handle field | Allocation, lifetime, global effects |
| F-be4e5c73e101 | Accepts the captured handle and returns the baseline | Internal reads, purity, application meaning |
| F-2dfa0238c98f | Accepts receiver and original adjustment when the strict comparison succeeds | Side effects, ownership behavior, failure behavior; result is unused |
| F-829bd0265134 | Accepts captured handle and replacement | Whether it stores, normalizes, rejects, or transforms the argument |
| Unresolved indirect operation at receiver-table offset 0x58 | Accepts receiver and branch-specific global operands; supplies the returned result | Concrete target, argument types, calling convention, internal effects |

Only the unreachable trailing region references F-870c4fcf7295, F-b07efd113d9d, F-b7af80e829ec, F-bcefe54b4f94, F-953474669c33, and F-4e292a3dd745. It also contains an additional direct dependency without a supplied opaque function ID and additional indirect operations. Their placement does not establish ownership or cleanup semantics.

## Acceptance examples

These are conditional static examples, not executed tests. “Replacement” means the computed value submitted to F-829bd0265134, not a verified stored value. Dependencies are assumed to return normally and preserve the validity required for subsequent accesses.

| Baseline | Adjustment | Invoke F-2dfa0238c98f? | Replacement | Terminal global arguments |
|---:|---:|---|---:|---|
| 50 | −10 | Yes | 40 | G-01, then G-02 by argument position |
| 50 | −50 | No | 0 | G-01, G-02 |
| 50 | −51 | No | 0 | G-01, G-02 |
| 50 | 0 | Yes | 50 | G-01, G-02 |
| 0 | 0 | No | 0 | G-01, G-02 |
| 50 | 10 | Yes | 60 | G-03, G-04 |
| 90 | 10 | No | 100 | G-03, G-04 |
| 95 | 10 | No | 100 | G-03, G-04 |
| 0 | 100 | No | 100 | G-03, G-04 |
| 20 | 101 | Yes | 121 | G-03, G-04 |
| 4,294,967,294 | 101 | Yes | 99 | G-03, G-04 |
| 4,294,967,295 | 101 | No | 100 | G-03, G-04 |

Additional acceptance conditions:

- With baseline 50 and adjustment −10, an optional-call change to external state does not cause baseline reacquisition. The submitted replacement remains 40, using the captured handle.
- With baseline 90 and adjustment 10, equality at the threshold skips F-2dfa0238c98f but does not skip F-829bd0265134 or terminal dispatch.
- If a preceding dependency changes G-01 and G-02, a nonpositive terminal dispatch uses their subsequently read contents, not entry-time values.
- If F-829bd0265134 changes G-03 to 7 and G-04 to 9, and those contents remain unchanged through argument evaluation, positive terminal dispatch receives receiver, 7, and 9 in that argument order. This does not establish which global is read first.
- If terminal dispatch returns 0x80000001, the function returns that unsigned 32-bit value, not a normalized Boolean.
- No ordinary signed adjustment selects the trailing receiver-local region under the displayed branches.

## Uncertainties

The following remain **documented_only**:

- Baseline meaning, valid domain, and caller-imposed adjustment limits.
- Dependency storage semantics, global mutations, reentrancy, ownership, and lifetime effects.
- Global operand declarations, widths, values, and meanings.
- Relative evaluation order among terminal global reads and receiver-table lookup.
- Concrete indirect targets and exact machine calling conventions.
- Signed conversion and downstream interpretation of unsigned results outside the signed range.
- Machine behavior for negating the most-negative signed adjustment.
- Outcomes for invalid addresses, invalid handles, abnormal dependency termination, or concurrent mutation.
- Whether the trailing region is genuinely unreachable machine behavior or reflects incomplete control-flow recovery.

The terminal operations are represented as calls despite explicit indirect-control-flow recovery qualifications. Their call and return behavior is established only at the supplied representation’s level.

## Evidence

**Established static evidence:** the supplied representation contains exhaustive signed-adjustment branches, provider and handle acquisition, a returned unsigned baseline, strict unsigned comparisons, conditional F-2dfa0238c98f invocations, unconditional replacement submissions, and terminal indirect operations whose results are returned. Each terminal operation uses a branch-specific pair of global data operands after F-829bd0265134 returns.

**Inferred intent:** ordinary-range inputs resemble bounded quantity adjustment. Unsigned threshold wraparound and zero-adjustment side effects prevent treating that interpretation as the full contract.

**Unknown:** dependency internals, global declarations, exact boundary instructions, and machine-level indirect control flow are not established by the supplied material.

No runtime tests have been performed.
