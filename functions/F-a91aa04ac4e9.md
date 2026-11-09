<!-- Trusted provenance: F-a91aa04ac4e9; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-a91aa04ac4e9

## Purpose

F-a91aa04ac4e9 conditionally processes an object-associated collection. It first consults an external service. If that service permits processing, it brackets the operation with two dependency calls, checks two object-related eligibility conditions, computes a processing argument, searches for one candidate, and processes that candidate through two independently advancing traversal states.

The function processes at most one selected candidate per invocation. It does not continue the candidate search after that candidate’s traversal finishes.

**Established behavior:** gating, dependency-call order, object-field accesses, candidate-search order, traversal scheduling, and explicit writes are visible in the supplied static representation.

**Inferred intent:** the bracketing calls may delimit an update operation, and the numeric preparation may compute a limit or budget. Those interpretations are not required by this contract.

**Unknown:** the collection’s abstract meaning, the external service’s role, and the numeric helpers’ mathematical semantics.

## Interface

### Input and return

The function accepts one object pointer and has no explicit return value.

The supplied representation uses 32-bit integer and pointer-sized accesses. This contract therefore describes the represented 32-bit layout, not a portable host-language structure.

The object must provide the following accessible locations when the relevant path executes:

| Byte offset | Access or use |
|---|---|
| `0x00` | Read as a dispatch-table pointer. |
| `0x08` | Address passed as an embedded collection or traversal context. |
| `0x40` | Address passed to the two bracketing dependencies. |
| `0x72` | One-byte status write on the eligible processing path. |
| `0x84` | Read as a 32-bit eligibility value and later as a multiplication operand. |

A callable slot at byte offset `0x0C` within the object’s dispatch table is invoked with the object pointer as an explicit argument.

The external service obtained at entry must expose a dispatch-table pointer at its start and a callable slot at byte offset `0x1A4`. The represented invocation of that slot has no explicit arguments.

### Candidate layout

A nonzero candidate returned by the search dependency must support these 32-bit reads:

| Byte offset | Meaning in this function |
|---|---|
| `0x0C` | Value used to derive an initial traversal position when the candidate’s classification is negative. |
| `0x20` | Signed classification controlling how that position is selected. |

The function allocates two distinct 16-byte local traversal-state areas. Their internal layouts are unknown. It also allocates a 32-bit local accumulator, initialized to zero before candidate searching, whose address is passed repeatedly to the processing dependency.

### Preconditions

There is no explicit validation of the input pointer, dispatch tables, callable slots, candidate pointers, or traversal outputs.

The caller and dependencies must provide valid storage and callable targets for every access reached. In particular, the service pointer is dereferenced without a null check.

Dependency termination, arithmetic validity beyond the represented machine widths, and the safety of returned traversal values are external preconditions. No exception, fault, or nonlocal-exit recovery is visible.

## Behavior

### 1. External gate

The first action is a call to F-87be4aa684d1 to obtain the external service pointer. The function then invokes that service’s slot at byte offset `0x1A4`.

If the slot returns any nonzero 32-bit value, the function returns immediately. On this path:

- No object field is explicitly read or written by this function.
- Neither bracketing dependency is called.
- No collection search or traversal occurs.

Effects performed by the service lookup or service method are not constrained by this observation.

### 2. Entering the processing region

If the service method returns zero, the function performs these actions in order:

1. Reads a 32-bit value through an external global pointer and passes it to F-d93d7d546d3b.
2. Saves that dependency’s 32-bit result for a possible later candidate-position calculation.
3. Calls F-4a19e3742733 with the address at object offset `0x40`.
4. Calls F-5b96767352e9 with the object pointer and saves its result as an unsigned 32-bit value.

If that unsigned result is zero, the function skips numeric preparation, status modification, candidate searching, and traversal. It proceeds directly to F-fd58bea73663 with the address at object offset `0x40`, then returns.

If the result is nonzero, the function reads the 32-bit field at object offset `0x84`. A zero field produces the same early completion through F-fd58bea73663.

The eligibility field is tested for equality to zero, not for positivity. Negative bit patterns are not excluded.

### 3. Preparing the processing argument

When both eligibility conditions are nonzero, the function executes a numeric-helper sequence. The sequence’s order is established, but its full mathematical interpretation is not.

The visible sequence is:

1. Call F-e79e242e842f with argument `8`.
2. Reread the object field at offset `0x84` and form its 32-bit product with that call’s result.
3. Pass the product to F-9e5e664c8ae7 and save its 64-bit result.
4. Call F-e79e242e842f with argument `9`.
5. Call F-9e5e664c8ae7 again. No explicit argument is shown for this invocation. Save its 64-bit result.
6. Invoke the object’s dispatch slot at byte offset `0x0C`, with the object pointer.
7. Call F-9e5e664c8ae7 a third time, again without an explicit argument shown.
8. Pass the low and high 32-bit halves of the third 64-bit result, followed by two external 32-bit operands, to F-c4b34006a13c.
9. Pass the low and high halves of that result and of the second saved 64-bit value to F-8a693c103d1c.
10. Pass the halves of the preceding result and of the first saved 64-bit value to F-8a693c103d1c.
11. Call F-baa332e06e32, with no explicit arguments shown, to obtain a signed 32-bit result.
12. Call F-e79e242e842f with argument `9`, then call F-d7be9cfb4000 with the signed result and the newly obtained value.

No floating-point interpretation, rounding rule, or association between omitted arguments and preceding return values is established here. The 64-bit values are explicitly split into low and high 32-bit words where stated.

The local accumulator is then initialized to zero.

The signed result from F-baa332e06e32 determines the next branch:

- **Nonnegative:** write byte value `0` at object offset `0x72`. Keep the result of the preceding F-d7be9cfb4000 call as the processing argument.
- **Negative:** call F-e79e242e842f with `9`, then with `8`; invoke the object’s slot at `0x0C` again; call F-d7be9cfb4000 with that method’s result and the 32-bit sum of the two selector results. Replace the processing argument with this new result, then write byte value `1` at offset `0x72`.

The first F-d7be9cfb4000 call occurs even on the negative branch, where its result is subsequently replaced. Its potential side effects remain observable.

### 4. Candidate search

The search starts with an unsigned 32-bit offset of zero and uses the address at object offset `0x08` as its context.

For each attempted offset:

- Call F-64f743a2929d with the context and the current offset.
- If that returns zero and the offset is nonzero, call it again with the 32-bit additive negation of the offset.
- Increment the unsigned offset by one.
- If either lookup returned a nonzero candidate, immediately select that candidate.
- Otherwise, stop searching when the saved eligibility result is less than or equal to the incremented offset, using an unsigned comparison. Continue at the next offset if it is greater.

Consequences:

- Offset zero is attempted once, without a duplicate “negative zero” lookup.
- At every nonzero magnitude, the positive lookup precedes the negated lookup.
- A successful positive lookup suppresses the corresponding negated lookup.
- Search terminates at the first nonzero candidate.
- With ordinary nonwrapping offsets and a bound of `N`, the attempted magnitudes are zero through `N − 1`.

Offsets and negation are 32-bit quantities. They must not be interpreted as arbitrary-precision signed integers. In particular, the high-bit-set midpoint is its own additive negation, so a failed lookup at that bit pattern can be followed by a second lookup with the same bit pattern.

If no candidate is found, the function calls F-fd58bea73663 and returns. The status byte has already been written on this path.

### 5. Selecting the initial traversal position

For the selected candidate, the function first reads the signed 32-bit field at candidate offset `0x20`.

- If that read is zero, it calls F-466606040074 with the object, candidate, and the saved result from F-d93d7d546d3b. The returned 32-bit value is the initial position.
- Otherwise, it reads the classification field again and tests whether that second value is less than one.
  - If so, the initial position is the 32-bit field at candidate offset `0x0C` minus one.
  - Otherwise, the initial position is zero.

With a stable classification field, these cases correspond to zero, negative, and positive classifications. The repeated read matters if memory can change between accesses; the function does not preserve the first classification value for the second test.

No explicit check rejects a negative position or validates the subtraction result.

### 6. Traversal and processing

The function initializes both 16-byte traversal states by calling F-7bfb6f538560 twice, in order, with:

- The respective local state address.
- The embedded context at object offset `0x08`.
- The selected candidate.
- The chosen initial position.

Both traversal directions initially remain active.

The function obtains an initial value from the first state through F-e161286c58b5 and immediately passes it to F-4cf039397475 together with:

- The object pointer.
- The address of the shared local accumulator.
- The prepared processing argument.

The return value from this initial processing call is ignored. In particular, a zero return does not deactivate either traversal direction.

Subsequent processing proceeds in rounds:

1. If the first direction remains active, obtain its next value through F-0463af715b9c, then process it through F-4cf039397475. That processing result becomes the first direction’s active state.
2. If the second direction remains active, obtain its next value through F-710579c35168, then process it through F-4cf039397475. That processing result becomes the second direction’s active state.

Any nonzero processing result keeps the relevant direction active; zero permanently deactivates it for this invocation. The first direction is serviced before the second in each round.

The same accumulator address and processing argument are used for every processing call. Changes to the accumulator by one call are available to later calls. This function does not otherwise inspect the accumulator.

Traversal values are not null-checked before processing.

When both directions become inactive, the function calls F-fd58bea73663 with the address at object offset `0x40`, then returns. No second candidate is searched.

## Side effects

The only explicit write into the input object is the one-byte status assignment at offset `0x72`. It occurs only after both eligibility checks succeed and numeric preparation completes.

Other direct writes are to local traversal storage, the local accumulator, and local saved values. Dependencies receive writable addresses and may produce additional effects not specified here.

F-4a19e3742733 and F-fd58bea73663 are paired on normal completion after the opening call returns. There is no visible guaranteed closing call if an intervening dependency fails to return, faults, or exits nonlocally.

There is no explicit allocation, deallocation, ownership transfer, or traversal-state destruction in this function. This does not establish that dependencies are allocation-free or that they preserve object, collection, or global state.

Local storage addresses supplied to dependencies remain valid only for this invocation. Whether any dependency retains those addresses is unknown.

## Dependencies

| Dependency | Established use |
|---|---|
| F-87be4aa684d1 | Obtains the service used for the entry gate. |
| F-d93d7d546d3b | Transforms a globally obtained value into a saved argument. |
| F-4a19e3742733 | Opens the bracketed region using the embedded address at `0x40`. |
| F-5b96767352e9 | Supplies the unsigned eligibility value and search bound. |
| F-e79e242e842f | Supplies selector-dependent values for arguments `8` and `9`. |
| F-9e5e664c8ae7 | Produces three 64-bit results during numeric preparation. |
| F-c4b34006a13c | Combines a split 64-bit value with two external words. |
| F-8a693c103d1c | Called twice with pairs of split 64-bit values. |
| F-baa332e06e32 | Supplies the signed value controlling the status branch. |
| F-d7be9cfb4000 | Produces the processing argument; called again on the negative branch. |
| F-64f743a2929d | Searches the embedded context by 32-bit offset. |
| F-466606040074 | Computes the initial position for a zero-classified candidate. |
| F-7bfb6f538560 | Initializes each local traversal state. |
| F-e161286c58b5 | Obtains the first state’s initial processing value. |
| F-0463af715b9c | Advances or obtains subsequent values for the first direction. |
| F-710579c35168 | Advances or obtains values for the second direction. |
| F-4cf039397475 | Processes a value and controls subsequent directional continuation. |
| F-fd58bea73663 | Closes the bracketed region on normal completion. |

The two dispatch-table calls are additional dependencies without supplied opaque function IDs. Their internal behavior is unknown.

## Acceptance examples

These are conditional static acceptance criteria, not executed tests.

1. **Gate rejects processing.** If the service method returns `1`, only the service lookup and gate invocation are required. No explicit status write or bracketing call follows.

2. **Zero search bound.** If the gate returns zero and F-5b96767352e9 returns zero, the opening and closing calls occur, but the field at `0x84` is not explicitly read for eligibility.

3. **Zero object eligibility field.** Given a nonzero bound and a zero field at `0x84`, the function closes normally without numeric preparation or status modification.

4. **Search misses with bound three.** Given successful eligibility and all-zero search results, the lookup arguments are `0`, `1`, `−1`, `2`, `−2`, interpreted as 32-bit patterns. No magnitude-three search occurs.

5. **Positive-side candidate wins.** If lookup at `1` returns a candidate, lookup at `−1` does not occur. Only that candidate is processed.

6. **Stable negative classification.** If the candidate classification is consistently negative and its field at `0x0C` equals `5`, both states receive initial position `4`; F-466606040074 is not called.

7. **Initial processing declines continuation.** Even if the initial F-4cf039397475 call returns zero, both traversal directions are still attempted in the first round.

8. **Independent stopping.** If the first direction’s processing returns zero and the second direction’s processing returns nonzero, later rounds skip the first direction and continue only the second.

9. **Negative numeric branch.** If F-baa332e06e32 returns a negative value, F-d7be9cfb4000 is called twice in total, the second result is used for processing, and the status byte becomes `1` before searching.

## Uncertainties

The following remain **documented_only**:

- Mathematical meanings and calling conventions of numeric helpers, especially invocations with no explicit arguments shown.
- Overflow behavior for the signed-looking 32-bit product, sum, and subtraction beyond their represented width; no validation is visible.
- The meanings of the external global input and the two external numeric operands.
- Whether either bracketing dependency implements locking, notifications, accounting, or another protocol.
- Whether dependencies modify globals, object fields, candidate storage, or collection structure.
- Ownership and lifetime of returned service, candidate, and traversal values.
- Whether traversal is guaranteed to terminate.
- Concurrency and memory-ordering guarantees.
- Whether dependency results encode errors. This function provides no separate error return or explicit error-reporting branch.

## Evidence

This contract is based solely on the supplied static function representation and dependency-ID map. It does not rely on dependency implementations or runtime observations.

Directly established evidence includes access widths and offsets, explicit call order, signed versus unsigned branch conditions, local-state sizes, search termination conditions, status writes, and independent traversal continuation flags.

Names such as “candidate,” “classification,” “direction,” and “processing argument” are descriptive labels, not recovered domain types or semantic guarantees.

No runtime tests have been performed.
