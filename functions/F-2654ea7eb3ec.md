<!-- Trusted provenance: F-2654ea7eb3ec; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-2654ea7eb3ec

## Purpose

F-2654ea7eb3ec dispatches an event against a mutable receiver and returns a Boolean indicating whether a supported handling path completed.

Three paths are established:

- Initial event type 7 or 13: extract and decode a command, apply one of two direction values, then invoke a subsequent receiver operation.
- A second event-type read yielding 19: evaluate primary and secondary matching conditions and, on success, invoke a receiver method.
- Initial event type 21: extract a payload, conditionally initialize a receiver-held helper reference, process the payload, and optionally apply a direction.

True does not necessarily mean that a direction was applied or that receiver state changed. Type-21 processing returns true even when its processing result is zero.

These are structural roles. Domain meanings such as navigation, selection, or keyboard handling are not established.

## Interface

### Arguments and representation

The function receives a receiver reference and an event reference.

The supplied representation uses 32-bit address-bearing values, 32-bit integer fields, and a Boolean result. All offsets below are byte offsets. The layouts describe only accesses visible in this function; they do not establish complete object definitions, allocation sizes, alignment rules, or a portable calling convention.

### Receiver layout

| Offset | Established width | Role |
|---|---:|---|
| 0x00 | 32 bits | Method-table reference used for indirect calls. |
| 0x40 | 32 bits | Primary match value. |
| 0x44 | 32 bits | Primary match identifier. |
| 0x98 | 32 bits | Secondary match identifier. |
| 0x9C | 32 bits | Secondary enable value; zero disables secondary matching. |
| 0xEC | 32 bits | Helper reference checked for initialization and later reread for processing. |

The method table is accessed at byte offsets 0xD4 and 0xA0. Both selected methods receive the receiver. The method selected at 0xA0 additionally receives a one-byte value. Their return values, if any, do not affect this function’s result.

### Event and payload layout

| Object | Offset | Established width | Role |
|---|---|---:|---|
| Event | 0x04 | 32 bits | Event type. |
| Event | 0x08 | 32 bits | Match value for type-19 handling. |
| Event | 0x0C | 32 bits | Match identifier for type-19 handling. |
| Extracted type-21 payload | 0x0C | 32 bits | Argument supplied to helper processing. |

The extracted payload reference is retained separately from the event reference. Whether these references alias is unknown.

### Shared inputs and width limits

Three shared inputs participate in the behavior:

- A primary alternate-match input.
- A secondary match input.
- An initialization configuration input.

The primary alternate-match and secondary-match inputs each feed a 32-bit integer assignment before comparison. Their **effective comparison values are therefore 32-bit integer values in the supplied representation**. This does not establish the widths, signedness, or access types of the underlying shared storage. Any conversion from that storage into the assigned integer is unspecified.

The initialization configuration is passed to F-1058afc38c04. Neither its underlying storage width nor the receiving argument’s type or width is established.

No declaration or explicit access type for these shared inputs is supplied. They must not be treated as three established 32-bit storage reads.

### Result, errors, and validity

The result is:

- **True** after a recognized directional command completes its two receiver calls.
- **True** after a successful primary or secondary match completes its selected method call.
- **True** after type-21 processing and any resulting directional call complete.
- **False** when no supported path accepts the event.

These outcomes assume that all reached accesses and calls complete normally. No separate error status, exception handling, recovery path, or rollback is visible.

There are no explicit null checks for the receiver, event, payload, method table, callable entries, allocation-like result, initializer result, or processing helper. Every reached access requires valid storage, and every reached call requires its dependency-specific preconditions. A zero value is not automatically safe merely because this function does not reject it.

## Behavior

### Dispatch and repeated type access

The first visible event access reads the 32-bit type at offset 0x04.

- Type 7 or 13 selects directional-command handling.
- Type 21 selects payload/helper handling.
- Every other initial value leads to a second read of the same field. Identifier matching proceeds only if that second read yields 19; otherwise the function returns false.

For stable event storage, the supported types are 7, 13, 19, and 21.

The two type reads are distinct in the supplied representation. If their values differ, the second read controls only entry into identifier matching. A second value of 7, 13, or 21 does not cause a return to the initial dispatch.

This structure does not establish volatile semantics, an atomic snapshot, synchronization, or safe concurrent mutation.

### Directional-command handling

For initial type 7 or 13:

1. F-682adfd5a561 receives the event reference and selector 0xA00.
2. F-11c1b7e655b3 receives the extraction result.
3. Its 32-bit result determines acceptance and direction.

| Decoded result | Behavior |
|---|---|
| 0x57 | Select the 32-bit value 0xFFFFFFFF, interpreted as −1. |
| 0x58 | Select the 32-bit value 1. |
| Any other value | Return false. |

For either recognized result, F-67baeb75ce7c receives the receiver and selected direction. After it returns, F-92bea7c30bd6 receives the receiver. The function then returns true.

Neither dependency’s return value controls this sequence. If the first call returns normally, the second is attempted.

For an unrecognized decoded value, extraction and decoding have already occurred. False therefore does not imply absence of dependency effects.

### Identifier matching

This path is entered only when the second type read yields 19.

The event identifier at offset 0x0C is read once and retained for both matching opportunities.

#### Primary match

The retained identifier is compared with receiver offset 0x44.

If the identifiers differ, primary matching fails without reading the primary event match value or receiver match value.

If they agree:

1. The event value at offset 0x08 is read.
2. The receiver value at offset 0x40 is read.
3. Those 32-bit values are compared.
4. Only if they differ is the shared primary alternate-match input evaluated into a 32-bit integer value.

Primary matching succeeds when the identifiers agree and the event value equals either the receiver value or that effective alternate value. An exact event/receiver value match does not evaluate the shared alternate input.

On success, the function obtains the receiver’s method-table reference, selects the entry at offset 0xD4, invokes it with the receiver, and returns true. Secondary matching is not attempted.

#### Secondary match

After primary failure, receiver offset 0x9C is read.

- Zero causes false.
- Nonzero permits comparison of the retained event identifier with receiver offset 0x98.

If those identifiers differ, the function returns false without reading the secondary event match value or evaluating the shared secondary-match input.

If they agree, the event value at offset 0x08 is read, followed by evaluation of the shared secondary-match input into a 32-bit integer value. Unequal effective values cause false.

Equal values cause this sequence:

1. F-5710a8628cba is called without visible arguments.
2. Its one-byte result is retained.
3. The receiver’s method-table reference and entry at offset 0xA0 are obtained.
4. The selected method receives the receiver and retained one-byte value.
5. The function returns true.

The method-table lookup occurs after F-5710a8628cba. No assumption that this dependency preserves the receiver or its method table is justified.

#### Precedence and retained state

Primary success takes precedence over all secondary conditions.

The event identifier is retained, not reread for secondary matching. The event match value can be read twice: once during a failed primary value comparison and again during secondary matching. These reads need not observe the same value if the storage changes.

The shared match inputs are evaluated only on their respective reached branches. Their effective integer values, not an assumed raw shared-storage representation, determine equality.

### Payload/helper handling

For initial type 21, F-682adfd5a561 receives the event reference and selector 0x3500. Its returned payload reference is retained.

Only after extraction returns does the function check receiver offset 0xEC. The initialization decision depends on that observed value, not the value at function entry. Extraction has no established receiver-state-preservation guarantee.

#### Conditional initialization

A nonzero helper reference skips initialization.

A zero helper reference causes:

1. F-8157a180d901 to receive the size value 0x34, decimal 52.
2. F-1058afc38c04 to receive that result, the numeric arguments 2 and 0, and the shared initialization configuration.
3. The initializer’s 32-bit result to be stored at receiver offset 0xEC.

The configuration is supplied after F-8157a180d901 returns; no earlier configuration snapshot is established. Its argument type and storage width remain unspecified.

Neither dependency result is checked. A zero initializer result is stored without rejection, retry, or an immediate false return. The stored value is the initializer result, not necessarily the preceding allocation-like result.

This store does not determine the helper argument ultimately passed to processing: two more dependencies execute before the field is read again.

#### Payload operations and late field reads

After the initialization decision and any resulting store:

1. F-c0a513a969e0 receives the retained payload reference and supplies a retained 32-bit result.
2. F-8bb3020950cd receives the same payload reference.
3. The payload’s 32-bit field at offset 0x0C and the current helper reference at receiver offset 0xEC are obtained.
4. F-aa672b15efb7 receives the helper reference, payload field, and retained result from F-c0a513a969e0.

Both field reads occur after F-8bb3020950cd returns. Their mutual evaluation order is not established.

The helper argument is whatever the late read observes. It need not equal the entry value, the post-extraction checked value, or the initializer result. F-c0a513a969e0 and F-8bb3020950cd have no established restriction against modifying that field.

The payload reference is retained across the calls, but its offset-0x0C field is read afterward. The payload must remain valid for that access. Preservation of its lifetime cannot be assumed from the supplied dependency information.

#### Processing result and arithmetic

The processing result is interpreted as signed 32-bit:

| Result | Directional behavior |
|---|---|
| Zero | No directional call. |
| Negative | F-67baeb75ce7c receives the receiver and 0xFFFFFFFF. |
| Positive | F-67baeb75ce7c receives the receiver and 1. |

Only zero and sign matter; magnitude is discarded. The minimum signed 32-bit value also selects −1. No negation, absolute value, or magnitude arithmetic is performed, so this selection introduces no signed-overflow boundary.

The path then returns true. It does not invoke F-92bea7c30bd6.

## Side effects

The only explicit direct write is the 32-bit store to receiver offset 0xEC when initialization is attempted.

Other effects are delegated. No dependency is established to preserve receiver fields, event storage, payload storage, method tables, or shared inputs. A false result does not guarantee unchanged state, and a true result does not guarantee a particular mutation.

No explicit deallocation or ownership-transfer operation appears. The size-52 request followed by initialization suggests allocation and construction, but those are inferred roles, not established ownership contracts. Allocation failure behavior, helper retention, replacement, destruction, and cleanup responsibility are unknown.

The payload is passed to multiple dependencies and subsequently dereferenced. Whether any dependency borrows, retains, consumes, releases, or invalidates it is unresolved.

If a reached access or call fails to complete normally, earlier effects—including a helper-field store—may remain. No rollback is visible.

## Dependencies

| Opaque dependency | Established interaction |
|---|---|
| F-682adfd5a561 | Receives the event and selector 0xA00 or 0x3500. Type-21 extraction precedes the helper check. |
| F-11c1b7e655b3 | Receives command extraction output; its 32-bit result is compared with 0x57 and 0x58. |
| F-67baeb75ce7c | Receives the receiver and exact 32-bit direction −1 or +1. |
| F-92bea7c30bd6 | Receives the receiver after recognized type-7/type-13 directional dispatch. |
| F-5710a8628cba | Supplies the one-byte argument for a successful secondary method call. |
| F-8157a180d901 | Receives size 52; its result feeds initialization. Allocation semantics are inferred. |
| F-1058afc38c04 | Receives the preceding result, 2, 0, and shared configuration of unspecified argument type; its 32-bit result is stored. |
| F-c0a513a969e0 | Receives the retained payload and supplies a retained 32-bit processing argument. |
| F-8bb3020950cd | Receives the retained payload before both late field reads. Lifetime and mutation effects are unknown. |
| F-aa672b15efb7 | Receives the late helper reference, payload field, and retained derived value; its signed 32-bit result controls direction. |

Two additional targets are selected dynamically through method-table offsets 0xD4 and 0xA0. Their opaque IDs are unavailable. Only the visible arguments, lookup order, and unused return values are established; broader signatures and effects remain unknown.

## Acceptance examples

These are conditional static acceptance criteria, not executed tests. Each assumes valid reached storage and normally returning dependencies.

1. **Negative command:** Initial type 7 and decoded result 0x57 cause direction 0xFFFFFFFF, followed by F-92bea7c30bd6, then true.

2. **Positive command:** Initial type 13 and decoded result 0x58 cause direction 1, the same subsequent receiver operation, and true.

3. **Rejected command:** Initial type 7 with any other decoded result returns false after extraction and decoding. This function directly invokes neither receiver operation on that branch.

4. **Primary exact match:** A second type read of 19, equal primary identifiers, and equal event/receiver values select method-table entry 0xD4. The shared alternate input is not evaluated; secondary matching is skipped.

5. **Primary alternate match:** Equal primary identifiers and unequal direct values still succeed if the effective 32-bit value obtained from the shared alternate input equals the event value. This criterion does not prescribe the shared input’s storage width or conversion.

6. **Secondary success:** Following primary failure, a nonzero enable field, equal secondary identifiers, and equality between the reread event value and the effective shared secondary value cause F-5710a8628cba followed by the method selected at 0xA0. That method receives the one-byte result.

7. **Secondary disabled:** Following primary failure, a zero enable field returns false without secondary identifier/value matching or the secondary dependency call.

8. **Initialization skipped:** For type 21, a nonzero helper field when checked after extraction skips initialization. A subsequent zero processing result still returns true, without directional dispatch or F-92bea7c30bd6.

9. **Initialization attempted:** A zero helper field at the post-extraction check causes the size-52 request, initialization with configuration obtained afterward, and storage of the initializer result. No particular configuration width is required by this contract.

10. **Entry value is insufficient:** A helper initially nonzero but changed to zero by extraction triggers initialization. A helper initially zero but changed to nonzero before the check skips it.

11. **Zero initializer and later replacement:** If initialization stores zero and a payload dependency replaces it before the late helper read, processing receives the replacement. If the late read instead observes zero, this function supplies zero; successful processing with zero is not guaranteed.

12. **Signed boundary:** A processing result of −2,147,483,648 selects exactly −1, not its magnitude. A positive result selects exactly +1. Neither case invokes F-92bea7c30bd6.

13. **Changed type between reads:** An unsupported initial type followed by a second type read of 19 enters matching. The same initial type followed by 21 returns false rather than entering payload handling.

14. **Unsupported stable type:** A stable type outside 7, 13, 19, and 21 is read twice and returns false without a listed dependency call.

## Uncertainties

The following remain **documented_only**:

- Domain meanings of types, selectors, match values, and directions.
- Exact Boolean return encoding and indirect-method signatures beyond visible arguments.
- Underlying storage widths, signedness, and access types of all three shared inputs.
- Conversions producing the two effective 32-bit shared match values.
- Initialization configuration argument type, width, and interpretation.
- Dependency effects, failure behavior, reentrancy, and preservation of shared or receiver state.
- Ownership and lifetime of allocation-like results, helper references, and payloads.
- Whether F-8bb3020950cd invalidates storage needed by the subsequent payload-field read.
- Whether zero is an acceptable helper argument to F-aa672b15efb7.
- Aliasing among receiver, event, payload, and shared storage.
- Synchronization, atomicity, and thread-safety requirements.
- Relative order of the two late field reads and machine-level accesses not established by the representation.

Allocation, construction, and directional meanings remain inferred intent rather than verified dependency contracts.

## Evidence

Established behavior comes from the supplied static function representation: branch constants, explicitly typed field accesses, receiver indexing, retained values, short-circuit conditions, dependency-call order, indirect method selection, the helper-field store and later reread, signed result classification, and Boolean return paths.

The shared inputs require a narrower conclusion than the object fields. Two feed 32-bit integer assignments, establishing their effective comparison values but not their underlying storage widths. The configuration appears as an initializer argument without a supplied declaration or explicit access type; neither its storage width nor argument width is established.

Extraction precedes the helper initialization check. Both payload dependencies precede the late helper and payload-field reads. These orderings support state-sensitive acceptance criteria without assuming dependency preservation or payload ownership.

Dependency bodies, complete type declarations, and runtime observations are unavailable. No runtime tests have been performed. All acceptance examples are conditional requirements derived from static evidence, not observed executions.
