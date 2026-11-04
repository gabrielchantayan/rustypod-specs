<!-- Trusted provenance: F-dee1b877e3f4; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-dee1b877e3f4

## Purpose

F-dee1b877e3f4 conditionally produces two independent, eight-byte output records using shared invocation-local working state. Each requested output has an associated dependency. A dependency return of zero permits that stage’s calculation and stores; a nonzero return selects an entry-captured failure token.

The stages execute in order and are not transactional. The first output is stored before the second dependency runs. A later failure causes no explicit rollback. Preservation of the first output nevertheless requires that the second dependency and other intervening actors not modify its storage.

The supplied static representation establishes control flow, explicit memory operations, and the assigned result widths. It does **not** establish the type or value of an operand in the first calculation. Consequently, that calculation’s intermediate widths, signedness, conversions, and overflow behavior cannot be specified unconditionally.

No runtime tests have been performed.

## Interface

### Inputs

| Input | Contract |
|---|---|
| First destination | Nullable address of storage for two consecutive 32-bit words. Nonnull requests the first stage. |
| Second destination | Nullable address of storage for two consecutive 32-bit words. Nonnull requests the second stage, unless the first stage fails. |
| Initial working value | Signed 32-bit value used to initialize one local working component. |
| Initial working index | Signed 32-bit value used to initialize the other local working component. |

“Value” and “index” identify roles in the calculations, not established application meanings.

Both dependencies receive writable addresses of the same two working components, in index-then-value order. Changes made by the first dependency are available to the second dependency without reinitialization.

A null destination suppresses that stage’s dependency call, calculation, and destination stores. It does not suppress the entry global reads or the other requested stage.

### Output layout

Each completed output occupies eight bytes:

| Offset | Width | Content |
|---|---:|---|
| 0 | 32 bits | Assigned signed 32-bit result |
| 4 | 32 bits | Result of shifting that assigned signed result right by 31 positions |

Under sign-propagating signed-right-shift semantics, the second word is zero for a nonnegative assigned result and all bits set for a negative assigned result.

This is a 32-bit result followed by a sign-derived word, not evidence of a general 64-bit calculation. In particular, unresolved intermediate arithmetic in the first stage does not change the established width of its assigned result or destination words.

Word order is established by the offsets. Byte order within each word and exact target signed-shift semantics are not independently established.

### Return value

The return is a 32-bit quantity:

- Zero when neither output is requested.
- Zero after every requested stage completes successfully.
- The entry-captured failure token when a requested dependency returns nonzero.

Dependency error values are tested for zero versus nonzero, not forwarded.

The failure token is not proven nonzero. A zero return alone therefore does not prove that requested outputs were written. This ambiguity is separate from memory faults, arithmetic outside the established contract, and abnormal dependency completion.

### Preconditions

Normal-return behavior requires:

- Every requested destination supports 32-bit stores at offsets 0 and 4.
- Applicable alignment and memory-access requirements are satisfied.
- Invoked dependencies return normally.
- Arithmetic evaluation and conversion to the assigned result have defined behavior under the applicable target rules.

No explicit validation of accessibility, alignment, bounds, overlap, or arithmetic range is established. No recovery contract is provided for invalid accesses or abnormal execution.

Destinations need not be distinct according to the visible checks. Overlap is handled only by the actual sequence of stores, not by buffering or overlap-specific logic.

## Behavior

### Entry state

The explicit entry operations occur in this order:

1. Read a signed 32-bit global multiplier.
2. Read a 32-bit global failure token.
3. Initialize the working value from its input.
4. Initialize the working index from its input.

Both globals are captured before any dependency call. Later explicit calculations use the captured multiplier; dependency-failure returns use the captured token.

A dependency may modify either global, but the function contains no explicit reread that would replace these captured values. This does not imply that dependencies preserve globals or other memory.

The two reads are separate. Their ordering does not establish an atomic snapshot or consistency under concurrent modification.

### First stage

A null first destination skips this stage entirely.

Otherwise, F-80e671e6c24b receives writable addresses of the working index and working value, in that order.

If it returns nonzero:

- The function returns the captured failure token immediately.
- The second dependency is not invoked.
- This function performs no explicit destination stores.
- Dependency side effects are not undone.

“No explicit destination stores” is not a promise that destination bytes remain unchanged. The dependency may reach those bytes through independently accessible memory.

If the dependency returns zero, the first calculation uses its post-call working state. Its mathematical structure is:

\[
\text{working value}
+
\bigl(\text{captured multiplier}
\times(\text{working index}+K)\bigr)
\times128
\]

Here \(K\) denotes an unresolved operand. Its numeric value and type are not established. A numeral embedded in an operand’s source-side label does not establish its value.

The expression’s outcome is assigned to a signed 32-bit result. That assigned result, rather than any wider intermediate, supplies both output words. The explicit stores occur in this order:

1. Assigned result at first-destination offset 0.
2. Assigned result shifted right by 31 at offset 4.

The first calculation does not assign its result back to either working component. Before any subsequent dependency effects, those components retain the values left by F-80e671e6c24b.

### Second stage

A null second destination skips this stage. If no earlier failure occurred, the function returns zero.

Otherwise, F-08fb88178805 receives the same working-component addresses in index-then-value order.

Its incoming state depends on the earlier branch:

- With no first-stage request, the components retain their input-initialized values.
- After first-stage success, they retain the values left by F-80e671e6c24b.
- After first-stage dependency failure, this call is unreachable.

When the first stage succeeds, both first-output stores precede this call. The call may nevertheless modify first-output storage through independently accessible memory.

If F-08fb88178805 returns nonzero, the function returns the captured failure token. It performs no explicit second-output stores and no explicit rollback of the first output.

If the dependency returns zero, the second calculation uses its post-call working components:

\[
\text{working value}
+
(\text{captured multiplier}\times\text{working index})
\times128
\]

This result is assigned back to the signed 32-bit working-value component. The function then stores:

1. The assigned working value at second-destination offset 0.
2. That value shifted right by 31 at offset 4.

The function then returns zero.

### Arithmetic widths, conversions, and boundaries

The working components, captured multiplier, and assigned results are represented as signed 32-bit quantities. This does not make every intermediate in both expressions unconditionally signed 32-bit.

**First calculation.** The unresolved type of \(K\) affects the addition to the working index. The resulting type can affect each subsequent multiplication and the final addition to the working value. Intermediate width, signedness, and arithmetic conversions therefore remain unresolved.

The first expression’s mathematical structure does not establish:

- Signed 32-bit evaluation throughout.
- Evaluation in a wider type.
- Unsigned or modular evaluation.
- Exact preservation of the mathematical value when assigned to the signed 32-bit result.

Once \(K\)’s type and conversion behavior are established, the applicable intermediate ranges can be determined. If all conversions preserve the intended operand values, every intermediate is representable in its actual evaluation type, and the final value is representable as signed 32-bit, the displayed mathematical expression determines the assigned result.

If separate evidence establishes signed 32-bit evaluation throughout, every intermediate must fit that range. Until then, applying a universal signed 32-bit intermediate limit to the first expression is unsupported.

The assignment’s 32-bit width is established even though conversion of an out-of-range expression value into that destination is not resolved here.

**Second calculation.** No unresolved operand is present. The represented arithmetic uses signed 32-bit working components and multiplier, with multiplication by 128 and no explicit widening. Its represented arithmetic stages are multiplier times index, that product times 128, and addition of the working value.

When each stage fits the signed 32-bit range, the mathematical expression determines the assigned result. A representable final mathematical result does not excuse an out-of-range intermediate.

Neither stage contains an explicit overflow check, saturation operation, or arithmetic-error return. Behavior outside the established evaluation and conversion conditions is documented_only, not an assumed wraparound contract.

## Side effects

### Explicit reads and writes

The function explicitly reads the multiplier and failure-token globals but does not explicitly write them. It initializes local working storage, exposes writable addresses of that storage to dependencies, and conditionally stores destination words. It does not explicitly read existing destination contents.

When both stages succeed, the explicit destination-store order is:

1. First destination, offset 0.
2. First destination, offset 4.
3. Second destination, offset 0.
4. Second destination, offset 4.

F-08fb88178805 runs between the second and third stores.

Each output is written using two distinct word stores. No atomic eight-byte update, synchronization, or rollback is established. An abnormal access can interrupt an update; the representation provides no recovery behavior for that case.

### Overlap and preservation

With identical destination addresses and successful completion of both stages, the second-stage stores replace both first-stage words. Without later interference, the shared region contains the second result.

For partial overlap, final contents depend on the addresses, stated store order, and any dependency or external writes. The function neither rejects overlap nor preserves overwritten first-output bytes.

If the second dependency fails, the absence of explicit second-output stores does not guarantee preservation of the first output. Such preservation requires that the dependency and other intervening actors leave that region unchanged.

### Ownership and lifetime

No explicit allocation, deallocation, or ownership transfer appears in this function. Caller-supplied destinations are used as output storage; their existing contents are not needed by the explicit calculations.

The dependencies receive addresses of invocation-local components. Their resource behavior, address retention, and access to unrelated memory are unknown. Passing these addresses does not establish a supported post-return lifetime or permission to use them after the invocation.

No claim is made that either dependency preserves globals, owns destinations, releases memory, or confines its effects to the supplied addresses.

## Dependencies

| Opaque ID | Invocation condition | Established call-site behavior |
|---|---|---|
| F-80e671e6c24b | First destination is nonnull | Receives writable index and value addresses. Zero permits first-stage calculation and stores. Nonzero causes immediate return of the captured failure token. |
| F-08fb88178805 | Second destination is nonnull and execution has not returned from first-stage failure | Receives the same addresses in the same order. Zero permits second-stage calculation and stores. Nonzero selects the captured failure token without explicit second-output stores. |

Accepted input domains, state transformations, error meanings, memory effects, resource ownership, and abnormal completion modes are not established.

The argument lists do not delimit all possible side effects.

## Acceptance examples

These are conditional static scenarios, not executed tests. Numeric output examples assume valid storage, normal dependency completion, and sign-propagating signed-right-shift semantics.

### Neither output requested

With both destinations null:

- Both entry globals are read and both working components initialized.
- Neither dependency is called.
- Neither calculation nor any destination store occurs.
- The return is zero.

### First dependency failure

With a nonnull first destination and a nonzero return from F-80e671e6c24b:

- The return is the captured failure token, not the dependency’s return value.
- F-08fb88178805 is not called, even if its destination is nonnull.
- This function performs no explicit destination stores.

Destination preservation additionally requires absence of dependency and external writes.

### Second output only

With the first destination null, a valid second destination, captured multiplier 2, and F-08fb88178805 returning zero with working value 5 and working index 3:

\[
5+(2\times3)\times128=773
\]

All represented intermediates fit signed 32-bit. The second destination receives 773 at offset 0 and zero at offset 4. The return is zero, and F-80e671e6c24b is not called.

### Negative second result

With only the second output requested, captured multiplier zero, and successful second dependency leaving working value −7 and working index zero:

- Offset 0 receives the signed 32-bit representation of −7.
- Offset 4 receives all bits set.
- The return is zero.

This example does not involve the unresolved first-stage operand.

### Conditional first-result determination

Suppose independent information establishes that \(K\) has value 1 and that the first expression evaluates with value-preserving signed 32-bit operations. With captured multiplier 2 and successful first dependency leaving working value 5 and working index 3:

\[
5+\bigl(2\times(3+1)\bigr)\times128=1029
\]

All intermediates fit the stated conditional evaluation model. The first output receives 1029 and a zero sign word.

This example does not establish \(K=1\), its type, or that evaluation model for the supplied representation.

### Unresolved first-stage evaluation

If the first dependency succeeds but \(K\)’s type remains unknown, success alone does not justify a signed 32-bit intermediate model or a concrete first-result value. In particular, a zero captured multiplier does not eliminate the need to account for evaluation of the index-plus-\(K\) subexpression.

### First success followed by second failure

With both destinations requested and first-stage calculation and stores completing:

- Both first-output stores precede F-08fb88178805.
- A nonzero second-dependency return selects the captured failure token.
- No explicit second-output stores or rollback occur.

The first output remains intact only if the dependency and other actors do not modify it.

### Shared state and captured globals

If the first dependency changes both working components, the second dependency receives those changed components, not restored arguments and not the calculated first result.

If either dependency changes the multiplier or failure-token global, later explicit operations still use the entry-captured values. Those global changes are not undone.

### Identical destinations

With identical valid destination addresses and both stages completing successfully, the second-stage stores replace both output words. Without later interference, the shared region contains the second output.

### Zero failure token

With an entry-captured failure token of zero and a requested dependency returning nonzero, the function returns zero while skipping that stage’s explicit output stores.

### Second-stage arithmetic boundary

With captured multiplier 1 and post-call working index 16,777,216, multiplication by 128 produces a mathematical value outside signed 32-bit range. Even if adding a negative working value would bring the final mathematical result back into range, the intermediate boundary prevents a defined numeric-output guarantee under the represented signed arithmetic model. No explicit range-error return is established.

## Uncertainties

The following remain documented_only:

- **Unresolved operand:** \(K\)’s value and type.
- **First-expression evaluation:** Intermediate widths, signedness, arithmetic conversions, and out-of-range assignment behavior affected by \(K\).
- **Arithmetic outside established ranges:** Target overflow behavior; no wraparound, saturation, or recovery is assumed.
- **Platform details:** Byte order, exact signed-right-shift rules, alignment, memory-fault behavior, and concurrency guarantees.
- **Global semantics:** Multiplier domain and failure-token meaning, including whether the token is nonzero.
- **Dependencies:** Transformations, independently accessible memory effects, retained references, ownership, and abnormal completion.
- **Output preservation:** Survival of an earlier output across later dependency execution requires a non-modification condition.
- **Application meaning:** Units, conversion domain, and semantic significance of the scale factor.

Intent inferred from the structure is limited to optional two-stage output production with shared mutable working state. No application-specific purpose is established.

## Evidence

Established static evidence supports the ordered global captures, input initialization, destination-controlled calls, shared working addresses, zero/nonzero branching, calculation structure, signed 32-bit assignments, ordered word stores, and return selection.

The first expression contains an operand whose type and value are not established. Its presence prevents extending the known signed 32-bit operand and assignment widths into an unconditional claim about every intermediate. The second expression lacks that unresolved operand and supports the separately stated signed 32-bit arithmetic contract.

The second dependency follows the first-output stores, but its memory effects are unknown. Therefore, absence of explicit rollback is established; unconditional preservation of the first output is not.

All examples are conditional deductions from the supplied static representation. No execution, trace, or runtime test has been performed.
