<!-- Trusted provenance: F-c0cca2da563c; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-c0cca2da563c

## Purpose

F-c0cca2da563c conditionally invokes an object-associated operation and returns a 32-bit Boolean-like result.

Its decision depends on an initial eligibility callback, a value obtained through one of two dynamically selected callbacks, a reference value obtained through another callback, and an unsigned 32-bit caller-supplied value. Two different permitted-gap limits apply, selected by a modular difference between the obtained value and the reference value.

The established behavior is a numerical gate followed by an indirect operation. The meanings of the values and the final operation are unknown. Interpretations such as addresses, positions, capacities, or timestamps are not part of this contract.

## Interface

### Inputs

The function receives:

1. **An object reference.** This is a pointer to an object whose first field supplies the base address of an indirect-call table.
2. **A candidate value.** This is an unsigned 32-bit integer.

No null check, object-type check, table-length check, or callback-address validation is visible.

### Object and dispatch layout

The supplied representation uses a 32-bit object/table address model. The object’s first field is read to locate the dispatch table. Relevant entries are identified by byte offsets from that table base:

| Table byte offset | Local opaque dependency label | Role in this contract |
|---|---|---|
| `0x08` | F-dep-01 | Initial eligibility callback |
| `0x94` | F-dep-02 | Reference-value callback |
| `0x98` | F-dep-03 | Selection probe and, conditionally, selected-value callback |
| `0xA0` | F-dep-04 | Alternative selected-value callback |
| `0x1A4` | F-dep-05 | Final operation |

These opaque labels identify dispatch roles within this document. They do not establish independently verified function identities, distinct implementations, or stable targets. Different entries may alias the same implementation, and callbacks may change the object or dispatch table.

Each indirect call receives the original object reference as its sole visible argument. The candidate value is not passed to any callback.

### Output

The visible return value is a 32-bit quantity with only two normal results:

- **0:** Initial eligibility failed, or the applicable numerical gate rejected the candidate.
- **1:** The final operation was invoked and returned normally.

The return value of the final operation is ignored. Consequently, a result of 1 does not establish that this operation succeeded according to any internal status it might return.

### Preconditions

For normal completion, every object-field read, table-entry read, and indirect call that the selected path reaches must be valid.

In particular, callbacks must leave enough valid state for subsequent reached accesses to proceed. This is a requirement on a normally completing execution, not an assertion that any callback actually preserves the object, table, globals, or memory ownership.

No memory-allocation, transfer, retention, or release contract is established by the wrapper.

## Behavior

### Arithmetic model

The numerical comparisons involving the selected value and the candidate are unsigned 32-bit comparisons.

Subtractions involving the selected value are evaluated modulo \(2^{32}\). A reference callback result represented as a signed 32-bit integer participates in the subtraction through its corresponding unsigned 32-bit value. Thus, its high bit does not cause the difference to be interpreted as signed.

Two quantities define the numerical gate:

- **Classifying difference:** selected value minus reference value, modulo \(2^{32}\).
- **Candidate gap:** selected value minus candidate value, considered only when the candidate is numerically no greater than the selected value.

For the candidate gap, the ordering check prevents unsigned underflow. The classifying difference has no such guard and may wrap.

### Initial eligibility

The function first obtains and invokes the callback at table offset `0x08`.

- If that callback returns zero, the function immediately returns 0.
- Any nonzero result allows evaluation to continue.

On this early-return path, none of the other listed callbacks is invoked.

### Selection of the value-producing callback

After initial eligibility succeeds, the function obtains and invokes the callback at offset `0x98` as a probe.

The probe result determines which table entry is subsequently used:

- A zero probe result selects offset `0xA0`.
- A nonzero probe result selects offset `0x98`.

The selected entry is then read and invoked. The unsigned 32-bit result of this invocation is the **selected value**.

When the probe is nonzero, the entry at offset `0x98` is therefore used for two calls: the probe call and a second value-producing call. The second return value, not the probe return value, supplies the selected value.

The entry is read again after the probe. The contract does not require the second call to reach the same implementation as the probe if the probe changed the object’s table pointer or table contents.

### Reference acquisition

After obtaining the selected value, the function obtains and invokes the callback at offset `0x94`.

Its 32-bit result supplies the reference value for classification. This callback occurs after both the probe and the selected-value call.

The selected value is retained across this reference callback; it is not reacquired afterward.

### Selection of the permitted gap

The function calculates the classifying difference using unsigned 32-bit subtraction.

The **smaller permitted gap** applies when the classifying difference is below `0x00124F80`.

The **larger permitted gap** applies when that difference is at least `0x00124F80`.

This boundary follows from the union of two adjacent ranges in the supplied comparison:

- Differences below `0x00124000`.
- Differences from `0x00124000` through `0x00124F7F`, inclusive.

There is no excluded point at `0x00124000`.

The applicable maximum candidate gaps are:

| Classifying difference | Maximum permitted candidate gap |
|---|---:|
| Below `0x00124F80` | `0x00002710` |
| At least `0x00124F80` | `0x0000EA60` |

Both maximum gaps are inclusive.

### Candidate comparison and rejection

If the candidate is greater than the selected value under unsigned comparison, the function does not calculate or enforce a candidate-gap limit. It proceeds to the final operation.

Otherwise, the candidate gap is calculated:

- Under the smaller-gap classification, the function returns 0 exactly when the gap exceeds `0x2710`.
- Under the larger-gap classification, the function returns 0 exactly when the gap exceeds `0xEA60`.

The intermediate boundaries `0x2000` and `0xE000` do not introduce additional rejected points. The comparisons collectively accept every gap from zero through the applicable maximum, without holes.

A rejection here occurs after the eligibility, probe, selected-value, and reference callbacks have already executed. Returning 0 does not roll back their effects.

### Final operation

If neither the initial eligibility check nor the numerical gate rejects execution, the function obtains and invokes the callback at offset `0x1A4`.

After that callback returns normally, the function returns 1.

No visible comparison is made against the final callback’s return value.

## Side effects

### Observable dispatch order

For a normally completing path that reaches the final operation, the visible order is:

1. Read the object’s table pointer, read the `0x08` entry, and invoke it.
2. Read the object’s table pointer, read the `0x98` entry, and invoke the probe.
3. Read the object’s table pointer, read the selected `0x98` or `0xA0` entry, and invoke it.
4. Read the object’s table pointer, read the `0x94` entry, and invoke it.
5. Perform the classification and candidate-gap decision.
6. Read the object’s table pointer, read the `0x1A4` entry, and invoke it.
7. Return 1.

An initial eligibility rejection stops after the first call. A numerical rejection stops after the reference callback and returns 0 without reading or invoking the final entry.

The supplied representation reevaluates the object’s table pointer for subsequent dispatches rather than establishing one table pointer for the entire operation. Mutations by earlier callbacks can therefore affect later dispatch targets.

### Writes and ownership

No explicit write to caller-provided storage, the object, or the table appears in the wrapper itself. No explicit allocation or release appears.

Every indirect callback can have effects not described by the supplied material, including changes to the object, table entries, shared state, or resource ownership. Such effects must not be assumed absent merely because the wrapper has no explicit writes.

There is no established rollback behavior on a result of 0.

### Errors and abnormal completion

No explicit error object, error code beyond 0, exception handling, or recovery path is visible.

Invalid pointers, invalid table entries, callback failures, callbacks that do not return, and memory-lifetime violations have no defined recovery behavior in the supplied material. The normal return contract applies only when all reached calls and accesses complete normally.

## Dependencies

| Opaque label | Required observation | Unspecified behavior |
|---|---|---|
| F-dep-01 | Its zero/nonzero result controls initial continuation. | Meaning of eligibility; side effects; ownership changes. |
| F-dep-02 | Its 32-bit result supplies the reference operand. | Units, range constraints, state preservation. |
| F-dep-03 | Its first result chooses the next entry; if nonzero, the entry at the same offset is read again for the selected-value call. | Whether repeated reads resolve to the same target; relation between the two results. |
| F-dep-04 | Supplies the selected value when the probe returns zero. | Semantic relationship to F-dep-03; side effects. |
| F-dep-05 | Is invoked only after the gate permits continuation; its return value is ignored. | Actual operation, success reporting, ownership and global effects. |

These dependencies are indirect and have no resolved external identities in the provided dependency map.

## Acceptance examples

These are conditional acceptance criteria derived from static evidence, not executed test results. Unless stated otherwise, all callbacks return normally and initial eligibility is nonzero.

### Initial rejection

If F-dep-01 returns zero, the required result is 0. No probe, selected-value callback, reference callback, or final operation is reached.

### Probe and selected value are separate

If the probe returns a nonzero value and the subsequent call through the freshly read `0x98` entry returns `0x00010000`, the selected value is `0x00010000`, regardless of the probe’s nonzero magnitude.

If the probe returns zero, the selected value instead comes from the subsequent call through offset `0xA0`.

### Smaller-gap boundary

Suppose the selected value is `0x00010000` and the reference value is also `0x00010000`. The classifying difference is zero, so the smaller limit applies.

- Candidate `0x0000D8F0` produces gap `0x2710`: invoke the final operation and return 1.
- Candidate `0x0000D8EF` produces gap `0x2711`: return 0 without invoking the final operation.
- Candidate equal to `0x00010000` produces zero gap: invoke the final operation and return 1.

### Larger-gap boundary

Suppose the selected value is `0x00200000` and the reference value is zero. The classifying difference selects the larger limit.

- Candidate `0x001F15A0` produces gap `0xEA60`: invoke the final operation and return 1.
- Candidate `0x001F159F` produces gap `0xEA61`: return 0 without invoking the final operation.

### Classification boundary

Suppose the selected value is `0x00200000` and the candidate is `0x001FD8EF`, producing gap `0x2711`.

- Reference `0x000DB081` produces classifying difference `0x00124F7F`. The smaller limit applies, so the function returns 0.
- Reference `0x000DB080` produces classifying difference `0x00124F80`. The larger limit applies, so the final operation is invoked and the function returns 1.

### Candidate greater than selected value

If the selected value is zero and the candidate is `0xFFFFFFFF`, unsigned comparison places the candidate above the selected value. The gap limits are bypassed, and the final operation is invoked.

This does not bypass the earlier eligibility, probe, selected-value, or reference calls.

### Wrapped classification

If the selected value is zero and the reference value is one, the classifying difference is `0xFFFFFFFF`, selecting the larger limit. This is unsigned modular behavior, not a negative difference.

### Ignored final status

If the gate permits continuation and the final operation returns a zero-valued status normally, the wrapper still returns 1.

## Uncertainties

The following remain **documented_only**:

- The domain meaning and units of all numerical values and constants.
- The concrete object type, complete layout, table extent, and calling-convention details beyond the visible argument and result use.
- The actual identities and contracts of the indirect callees.
- Whether callbacks mutate the object, replace its table, alter globals, allocate, retain, or release resources.
- Whether any external invariant rules out wrapped classification or candidates above the selected value.
- Thread-safety, reentrancy, synchronization, and concurrent mutation behavior.
- Platform-specific consequences of invalid memory or abnormal callback completion.

The return values suggest a Boolean gate, and the last callback suggests an operation authorized by that gate. Those are inferred roles, not established domain semantics.

## Evidence

The contract is based solely on the supplied static function representation.

Established evidence includes dispatch offsets, callback order, repeated table lookup after the probe, the two normal return values, unsigned comparison and subtraction behavior, inclusive gap boundaries, and the ignored final callback result.

The consolidated classification threshold and maximum-gap rules are algebraic consequences of the supplied comparisons; they do not rely on assumed application intent.

No runtime tests have been performed. No callback implementation, external ownership contract, or independent implementation identity was available.
