<!-- Trusted provenance: F-7fa5bdb0d753; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-7fa5bdb0d753

## Purpose

F-7fa5bdb0d753 performs gate-controlled bookkeeping followed by an optional indirect invocation.

Its established behavior is:

1. Invoke F-f4c5ae40135d and retain its result.
2. Capture the address held in a shared-record locator, including when the gate result is zero.
3. If the gate result is nonzero, perform a four-byte decrement using further references to that locator.
4. Inspect an object pointer through the **previously captured address**.
5. If the object pointer and the object’s first integer are both nonzero, invoke a target held in separate storage.

The address used for object-pointer inspection is not necessarily the address used for either side of the decrement. Their equivalence depends on the locator remaining stable across the relevant evaluations.

“Gate,” “record,” and “object” describe observable roles. Reference counting, destruction, callback ownership, and resource release are not established purposes.

## Interface

### Explicit inputs and output

The represented interface has no explicit parameters and no explicit return value.

F-f4c5ae40135d produces an integer result that is tested against zero. The conditional indirect invocation has no displayed arguments, and this function does not consume a result from it.

The function does not report success or failure through an explicit return value. Its observable behavior consists of dependency execution, external-state reads, a conditional memory write, and a possible indirect invocation.

### External state and layout

| State | Established use |
|---|---|
| Shared-record locator | Holds a record address. It is referenced once for the saved address after the gate returns and again in the decrement expression. |
| Record offset 0 | Four-byte slot interpreted as an object address. This slot is read through the saved record address. |
| Record offset 8 | Four-byte word used by the decrement. Its source and destination are addressed through references to the locator within that expression. |
| Referenced object offset 0 | Integer represented as four bytes and tested against zero. |
| Indirect-target storage | Supplies the target for the conditional invocation; it is distinct from the object’s tested integer. |

The record is indexed in four-byte units, placing the decremented word at byte offset 8. No explicit access to record offset 4 appears.

The supplied representation uses a four-byte pointer-bearing record slot. This supports a 32-bit address representation for that slot, but does not independently establish the complete machine ABI, pointer validity rules, alignment, or byte order.

The gate result and the referenced object’s first value are represented as integers. The supplied representation supports 32-bit integer interpretation; their application-level meanings are unknown.

### Safe-execution preconditions

If the gate returns normally, the locator must support the address capture that follows, even if the gate result is zero. A zero result does not require the captured address to designate readable record contents, because this body does not then dereference it.

On the nonzero path, safe ordinary completion additionally requires:

- The locator references used by the decrement to yield suitable addresses for the source read and destination write at offset 8.
- The saved address to permit a four-byte read at offset 0.
- A non-null object address read from that slot to permit a four-byte read at object offset 0.
- A conditionally selected indirect target to be callable under the actual execution environment.
- Relevant storage to remain valid for its individual accesses.

These are execution preconditions, not checks implemented by this function. The saved record address and the decrement addresses are not explicitly checked for null or validity.

## Behavior

### Gate execution and address capture

F-f4c5ae40135d is invoked first.

If that dependency returns normally, the function captures the address currently held in the shared-record locator. This capture precedes the decision to return for a zero gate result.

Consequently:

- The function does not retain a record address from before the gate call.
- Changes made by the gate to the locator can affect the captured address and subsequent decrement references.
- A zero gate result still reaches the locator capture.
- If the gate does not return normally, none of the later operations is reached through the displayed flow.

When the gate result is zero, the function returns after the capture. It performs no explicit record-field read or write, no object-field read, and no indirect invocation.

This is not an overall no-side-effect guarantee: the gate has already executed, and its effects are unknown.

### Decrement through subsequent locator references

Every nonzero gate result selects the active path. The sign and magnitude of that result do not otherwise influence the visible behavior.

The next statement computes a four-byte word decremented by one and stores the result at record offset 8. Crucially, this statement does **not** use the previously saved record address. Both its source-field expression and its destination-field expression refer to the shared-record locator again.

The contract therefore distinguishes three address roles:

| Address role | Origin | Use |
|---|---|---|
| Saved record address | Locator capture immediately after the gate returns | Later read of the object-pointer slot at offset 0 |
| Decrement-source address | Locator reference in the decrement’s source expression | Read of the word at offset 8 |
| Decrement-destination address | Locator reference in the decrement’s destination expression | Write of the decremented word at offset 8 |

With a stable locator, all three addresses designate the same record. Without that stability, the representation does not justify collapsing them into one selected record. In particular, the saved address may differ from the decrement addresses, and the decrement’s source and destination address evaluations may resolve differently.

The representation establishes the surrounding statement order:

- Saved-address capture precedes the decrement statement.
- The decrement statement precedes the object-pointer read through the saved address.

It does **not** establish an evaluation order between the source-side and destination-side locator references within the decrement expression. A contract must not assign one of those locator evaluations before the other. Nor does the representation establish the exact number of machine-level locator loads.

The resulting write depends on the source word read by the decrement, not necessarily on the previous contents of the destination word if those addresses differ.

The update does not depend on whether the later object pointer is null, whether the object’s first integer is zero, or whether the indirect invocation succeeds.

### Arithmetic width and boundaries

The explicitly updated storage is four bytes, and the represented operation is decrement by one.

At the 32-bit word level, this is a decrement modulo \(2^{32}\): a source word of zero yields an all-ones result, while an all-ones source word yields the word with bit pattern \(2^{32}-2\). There is no visible saturation, lower-bound guard, or zero-specific branch.

This describes the represented fixed-width word operation, not an independently established application-level signed-counter contract. The underlying field’s signedness and the language-level treatment of signed overflow are not supplied.

No atomic read-modify-write guarantee follows from the single expression. Synchronization and behavior under concurrent mutation are unresolved.

### Object-pointer inspection through the saved address

After the decrement statement, the function reads the four-byte slot at offset 0 using the **saved record address** captured before that statement.

It does not obtain this inspection address by reloading the locator at this point. Therefore, a changed locator does not by itself redirect this particular pointer read.

If the pointer read from the saved record is null:

- No object-field read occurs.
- The indirect dependency is not invoked.
- The function returns after the decrement has already occurred.

If the pointer is non-null, the function reads the integer at the referenced object’s offset 0.

If that integer is zero, no indirect invocation occurs. If it is nonzero, the invocation condition is satisfied. Positive and negative represented integers both satisfy the nonzero test.

The function does not explicitly write the object-pointer slot or the object’s first integer. Nevertheless, the earlier decrement could affect a later-read location if the relevant storage aliases. Non-aliasing is not established.

### Conditional indirect invocation

When the object pointer and its first integer are both nonzero, the function invokes the target supplied by the separate indirect-target storage.

The tested object integer is a condition for invocation. It is not established as the invoked address. Neither the object pointer nor the decremented word is shown as an explicit argument.

The invocation follows the decrement and object inspection. No local target-null check, target-validity check, or recovery mechanism is visible.

The supplied representation identifies uncertainty in recovery of the indirect control transfer. The conditional target use is represented as an invocation, but the exact machine-level call-versus-transfer mechanism remains unresolved.

If the indirect dependency returns normally, the function then returns without further explicit record accesses or updates.

## Side effects

### Explicit effects

The only explicit data write is the four-byte decrement-result store at offset 8 through the decrement-destination address. It occurs only for a nonzero gate result.

The body contains no explicit allocation, deallocation, pointer clearing, target reset, increment, rollback, logging, or error reporting.

A zero gate result still entails the gate call and subsequent locator capture, but no explicit record dereference.

### Dependency effects and ownership

F-f4c5ae40135d executes on every invocation. Its effects are not constrained by the supplied body. In particular, it may change the locator, records, referenced objects, or target storage before the local accesses occur.

The indirect dependency executes only after the local decrement and the two nonzero object conditions. It may itself modify shared state, transfer ownership, release storage, or fail to return; none of those behaviors is established or excluded here.

The function performs no explicit ownership acquisition, transfer, or release. A decrement alone does not establish reference-count semantics. No dependency is assumed to preserve globals or memory lifetimes.

### Failure behavior

No explicit error result or local recovery path is provided for inaccessible storage, invalid addresses, or an invalid indirect target.

If a later read or invocation fails after the decrement store, no rollback is visible. The store precedes those later operations in the represented flow; its visibility and persistence in a faulting execution depend on platform semantics.

## Dependencies

| Opaque function ID | Established interaction | Unresolved behavior |
|---|---|---|
| F-f4c5ae40135d | Invoked once before the saved-address capture. Its returned integer controls the zero/nonzero branch. | Result meaning, state changes, ownership effects, exceptional exits, and whether it returns normally. |
| F-indirect-1 | Conditional invocation through separate target storage after the decrement, a non-null object pointer, and a nonzero object integer. | Target identity, complete ABI, hidden inputs, state changes, ownership effects, and return behavior. |

F-indirect-1 is a document-local opaque identifier for the unresolved target, not a recovered direct dependency.

Neither dependency is assumed to preserve the locator, records, object storage, or target storage.

## Acceptance examples

These are conditional acceptance cases derived from static evidence, not runtime results. Unless otherwise stated, they assume accessible storage, normal dependency returns, no relevant aliasing, and stability during the specified accesses.

| Conditions | Required behavior |
|---|---|
| The gate returns zero. | Capture the locator’s address, then return. Do not dereference record fields or invoke F-indirect-1. The gate may already have produced effects. |
| The gate returns nonzero; the locator remains stable; the offset-8 word is 9; the offset-0 pointer is null. | Write 8 to offset 8, then read the null pointer through the saved address. Do not read an object or invoke F-indirect-1. |
| The gate returns nonzero; the locator remains stable; the word is 9; the object pointer is valid and its first integer is zero. | Write 8 before object inspection. Read the object integer, but do not invoke F-indirect-1. |
| The same conditions hold, except the object integer is 1. | Write 8, inspect the object, then invoke the separately stored target. |
| The gate result and object integer are negative represented integers. | Both pass their respective nonzero tests. With the remaining preconditions satisfied, perform the decrement and indirect invocation. |
| The decrement source word is zero. | Produce an all-ones 32-bit result at the decrement destination. Object inspection still follows independently. |
| The gate changes the locator from one record to another and it remains stable thereafter. | The post-call capture and decrement references select the new record. No pre-call record address is retained by this body. |
| The saved address designates one record, while both decrement address evaluations resolve to a different record. | Decrement the latter record’s offset-8 word, but read the object pointer from the former record’s offset-0 slot. The latest locator value does not replace the saved address for inspection. |
| The decrement-source evaluation resolves to a record containing 12 at offset 8, and the destination evaluation resolves to another record containing 90 there. | Store 11 in the destination word, not 89. These conditional address resolutions do not imply an evaluation order between the locator references. |
| The saved record address is null on the nonzero path. | No local guard prevents the later offset-0 read through that saved address. A valid decrement address does not make the inspection safe. |
| The object is valid and its first integer is nonzero, but the separate target is invalid. | Attempt the indirect invocation after the decrement and object read. No local recovery is established. |

The differing-address examples describe the consequences **if those address resolutions occur**. They do not promise that unsynchronized concurrent mutation has defined semantics or that a particular machine instruction sequence produces those resolutions.

Changing only the indirect target cannot enable invocation when the inspected object integer is zero. Conversely, a nonzero object integer does not determine the target address.

## Uncertainties

### Established static evidence

The representation establishes:

- The unconditional gate call.
- The subsequent saved-address capture, including on the zero-result path.
- Separate locator references in the decrement’s source and destination expressions.
- The decrement statement before object inspection.
- Use of the saved address for the offset-0 pointer read.
- A guarded object-integer read and conditional indirect target use.

The distinction between these address sources is established dataflow, not merely a concurrency concern.

### Inferred intent

Guarded bookkeeping followed by conditional dispatch is a reasonable descriptive interpretation. Reference counting, destruction, ownership release, and callback lifecycle management remain unsupported interpretations.

### documented_only unknowns

- Complete object layouts, field meanings, and total allocation sizes.
- Application-level signedness and source-language arithmetic rules.
- Complete ABI, alignment, byte order, and hidden argument passing.
- Exact machine-level locator-load count.
- Evaluation order within the decrement expression beyond the data dependencies needed to compute and store its result.
- Volatile, device-memory, atomic, and synchronization properties.
- Behavior under concurrent mutation.
- Aliasing among the locator, records, referenced object, and target storage.
- Exact indirect-transfer mechanism.
- Dependency effects, global preservation, ownership rules, and lifetime guarantees.
- Platform fault handling and nonlocal control effects.

These unknowns do not permit replacing the distinct saved, source, and destination address roles with a single assumed record.

## Evidence

This contract uses only the supplied static function representation and opaque dependency mapping.

The decisive evidence is the separation between the post-gate address capture, the later decrement expressions that refer to the locator again, and the subsequent object-pointer read through the saved address. The supplied representation also establishes the field widths, offsets, branch conditions, and separate indirect-target storage.

Recovery of the indirect transfer is explicitly incomplete, so its precise low-level mechanism remains documented_only.

No runtime tests, execution traces, binary inspection, or implementation changes have been performed.
