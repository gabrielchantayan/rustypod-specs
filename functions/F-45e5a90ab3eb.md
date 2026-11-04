<!-- Trusted provenance: F-45e5a90ab3eb; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-45e5a90ab3eb

## Purpose

F-45e5a90ab3eb conditionally processes an input record against state associated with a receiver object. It lazily initializes that state, obtains a token from the input, performs two state-dependent operations, and may notify the receiver through an indirect callback.

The established behavior is a control-flow and data-flow contract. The meanings of the token, associated state, computed result, and notification are not established. In particular, describing this function as insertion, lookup, comparison, or event delivery would require evidence not supplied here.

The normal return value distinguishes an initial suppression condition from all other normally completing paths:

- **0:** a receiver flag prevented processing.
- **1:** processing reached a normal completion, whether or not a callback occurred.

A return of 1 does not independently establish that an allocation succeeded, a record was accepted, or a callback reported success.

## Interface

### Inputs

The function receives two machine-level arguments:

1. **Receiver:** an address of an object with accessible 32-bit fields and an indirect-dispatch structure.
2. **Input record:** a 32-bit address-like argument passed to two dependencies and used as the base address for a 32-bit field read.

The supplied representation supports the following receiver layout:

| Byte offset | Width | Established use |
|---|---:|---|
| `0x00` | 32 bits | Address of an indirect-dispatch table, used only on the notification path. |
| `0x48` | 32 bits | Flags word. Bit mask `0x01000000` suppresses this operation. |
| `0x108` | 32 bits | Associated-state handle. Zero triggers initialization. |

On the notification path, a callable address is obtained from byte offset `0x15C` within the table addressed by the receiver’s first word. The source does not establish the table’s other entries or the receiver’s complete size.

The input record has one directly accessed field:

| Byte offset | Width | Established use |
|---|---:|---|
| `0x0C` | 32 bits | Opaque value supplied to F-aa672b15efb7. |

The input record may have additional layout requirements imposed by dependencies. Those requirements are unknown.

### Output

The observable return is a 32-bit value, either 0 or 1, if execution returns normally.

A separate signed 32-bit value is used for notification. It is not the function’s return value. Depending on a dependency result, this notification value is either preserved unchanged or normalized to −1 or +1.

### Preconditions and validity

The function performs no explicit null, bounds, allocation-failure, or dispatch-target validation.

Required accessibility is path-dependent:

- Reading the receiver’s flags is required even on the suppressed path.
- Access to the associated-state field is required only when processing is not suppressed.
- Input-record validity is required by the reached dependencies and by the later read at offset `0x0C`.
- The dispatch table and callable slot need to be valid only if notification is reached.

Dependencies may impose additional preconditions. Their absence from the supplied evidence must not be interpreted as permission to use arbitrary handles or addresses.

## Behavior

### 1. Initial suppression

The function first reads the 32-bit receiver flags at offset `0x48`.

If any bit selected by `0x01000000` is set, it returns 0 immediately. On this path, the function itself:

- Does not inspect the associated-state field.
- Does not inspect the input record.
- Does not invoke any dependency.
- Does not write receiver state.
- Does not perform indirect notification.

Other bits in the flags word do not affect this decision.

The suppression check is performed once. The function does not explicitly recheck the flag after any dependency call.

### 2. Conditional initialization

If processing is not suppressed, the function reads the 32-bit associated-state field at receiver offset `0x108`.

If that value is nonzero, initialization is skipped.

If that value is zero, the function performs the following ordered actions:

1. Calls F-8157a180d901 with the numeric argument `0x34`, or 52.
2. Calls F-1058afc38c04 with:
   - The 32-bit result from F-8157a180d901.
   - The numeric argument 4.
   - The numeric argument 1.
   - The value of an opaque global datum.
3. Stores the 32-bit result from F-1058afc38c04 into receiver offset `0x108`.
4. Calls F-991ae23131ae with that same returned result and the numeric argument `0x20`, or 32.

The store occurs before F-991ae23131ae is called. That dependency therefore runs after the newly returned handle has been published in the receiver.

The configuration call receives the result retained from F-1058afc38c04, not a fresh read of the receiver field. Conversely, subsequent processing obtains its state arguments through later reads of the receiver field. This distinction matters if dependencies modify receiver state.

There is no explicit rejection of a zero result from either initialization dependency. A zero result from F-1058afc38c04 is still stored and passed to F-991ae23131ae. No retry or alternative initialization path is present.

Calling F-8157a180d901 with 52 suggests a storage request, but allocation semantics, units, and failure behavior are not established by this function alone.

### 3. Input-dependent operations

After initialization has either completed or been skipped, the function:

1. Calls F-c0a513a969e0 with the input-record argument and retains its 32-bit result as an opaque token.
2. Calls F-8bb3020950cd with the same input-record argument.
3. Reads the input record’s 32-bit field at offset `0x0C` and obtains the current associated-state handle from the receiver.
4. Calls F-aa672b15efb7 with the associated-state handle, the input field value, and the retained token. Its result is subsequently interpreted as a signed 32-bit integer.
5. Obtains the associated-state handle again and calls F-be0ad8f71bac with that handle and the retained token.

The token is obtained before F-8bb3020950cd runs, while the direct input-field read occurs afterward. A mutation by F-8bb3020950cd can therefore affect the field supplied to F-aa672b15efb7 without changing the already retained token.

The two state-dependent calls need not receive the same handle. The receiver field is read for each call, and the first call or other intervening effects may change it.

The relative order of argument-value reads within a single call preparation is not established. The order of the dependency calls themselves is established.

F-be0ad8f71bac is called even when F-aa672b15efb7 returned zero. The later decision to omit notification does not bypass that call.

### 4. Notification decision and normalization

Let **result** denote the signed 32-bit return from F-aa672b15efb7. Let **selector** denote the return from F-be0ad8f71bac, interpreted only as zero versus nonzero.

The decision is:

| Selector | Result | Action |
|---|---|---|
| Zero | Zero | Return 1 without notification. |
| Zero | Negative | Notify using the unchanged result. |
| Zero | Positive | Notify using the unchanged result. |
| Nonzero | Zero | Return 1 without notification. |
| Nonzero | Negative | Notify using −1. |
| Nonzero | Positive | Notify using +1. |

Thus, zero results always suppress notification. The selector controls whether a nonzero result retains its magnitude.

Normalization is based on signed comparisons. It does not negate a negative result or compute an absolute value. In particular, the minimum signed 32-bit value can be normalized to −1 without requiring a representable positive counterpart.

### 5. Indirect notification and conditional follow-on call

When notification is required, the function resolves the receiver’s dispatch table and its callable entry at byte offset `0x15C`.

It invokes that callable with:

- The receiver address.
- The selected signed 32-bit notification value.
- The numeric argument 1.

If the callable returns zero, no follow-on call occurs.

If it returns any nonzero value, the function invokes a further dependency with the same notification value. That dependency is identified here as F-unresolved-01; this is a document-local opaque identifier, not a supplied published mapping.

If all reached calls return normally, F-45e5a90ab3eb then returns 1.

No recovery path is established for a dependency or indirect callback that faults, does not return, or otherwise prevents ordinary continuation.

## Side effects

### Directly established effects

The function has one explicit receiver-state write: storing the initialization result at offset `0x108` when the initial associated-state value is zero.

Other directly established observations include:

- Reading the initial flags and associated-state fields.
- Reading an opaque global datum on the initialization path.
- Reading the input field after F-8bb3020950cd.
- Reading the associated-state field separately for the two state-dependent calls.
- Reading the dispatch-table address and callable entry when notification is required.

There is no explicit direct write to the input record, flags word, dispatch table, or opaque global datum.

### Dependency effects and ownership

Every reached dependency may have additional effects that are not established here. In particular, this contract does not assume that dependencies preserve globals, preserve receiver fields, leave the input record unchanged, or retain the validity of earlier addresses.

The function contains no explicit release operation for the value returned by F-8157a180d901, the associated-state handle, the token, or the input record. This does **not** establish a leak or an ownership transfer: a dependency could retain, consume, or release resources.

The function provides no explicit synchronization or protection against concurrent initialization. Whether concurrent or reentrant use is valid is unknown.

## Dependencies

| Opaque ID | Established call contract at this call site |
|---|---|
| F-8157a180d901 | Receives 52; its 32-bit result becomes the first argument to F-1058afc38c04. |
| F-1058afc38c04 | Receives the preceding result, 4, 1, and an opaque global value; returns the value stored as associated state. |
| F-991ae23131ae | Receives the newly stored initialization result and 32; its return is not used. |
| F-c0a513a969e0 | Receives the input-record argument; returns the retained 32-bit token. |
| F-8bb3020950cd | Receives the input-record argument after token acquisition; its return is not used. |
| F-aa672b15efb7 | Receives current associated state, the input field, and the token; returns the signed result controlling notification. |
| F-be0ad8f71bac | Receives freshly read associated state and the token; its zero/nonzero result controls normalization. |
| F-indirect-01 | Document-local opaque identifier for the receiver’s callback at dispatch-table offset `0x15C`; receives receiver, notification value, and 1. |
| F-unresolved-01 | Document-local opaque identifier for the unmapped follow-on dependency; receives the notification value only when F-indirect-01 returns nonzero. |

Names such as “token,” “selector,” and “associated state” describe roles in this function, not independently verified dependency semantics.

## Acceptance examples

These examples are conditional static acceptance criteria, not reports of executed tests. Unless stated otherwise, all reached dependencies return normally and all required accesses remain valid.

1. **Suppression takes precedence.**  
   Given a receiver flags word containing `0x01000000`, the function returns 0 without calling dependencies, even if its associated-state field is zero. The input record is not directly accessed.

2. **Existing state skips initialization.**  
   Given an unset suppression bit and a nonzero associated-state field, none of F-8157a180d901, F-1058afc38c04, or F-991ae23131ae is called. Input processing still proceeds.

3. **Initialization publishes before configuration.**  
   Given an initial zero associated-state field and an initialization result H, the receiver field is written with H before F-991ae23131ae receives H and 32. If H is zero, there is no explicit early return or retry.

4. **Zero result still reaches the selector dependency.**  
   Given that F-aa672b15efb7 returns zero, F-be0ad8f71bac is still called. Regardless of its return, no notification occurs and the function returns 1.

5. **Zero selector preserves magnitude.**  
   Given a result of −7 and a zero selector, F-indirect-01 receives −7. Given a result of +7 under the same condition, it receives +7.

6. **Nonzero selector normalizes sign.**  
   Given a result of −7 and any nonzero selector, F-indirect-01 receives −1. Given a result of +7, it receives +1.

7. **Signed lower boundary is not negated.**  
   Given a result of −2,147,483,648, a zero selector preserves that value; a nonzero selector produces −1.

8. **Input preparation can affect the later field read.**  
   If F-8bb3020950cd changes the input field at offset `0x0C`, the subsequent call uses the post-call field value, while retaining the token acquired earlier.

9. **Follow-on call depends on callback return.**  
   For any reached notification, a zero callback return suppresses F-unresolved-01. Any nonzero callback return causes that dependency to receive exactly the notification value.

## Uncertainties

The following are **documented_only**:

- Semantic types and ownership rules for handles, tokens, the input field, and the opaque global datum.
- Whether the initialization dependencies allocate, construct, configure, register, or perform unrelated operations.
- Dependency failure conventions, including whether zero is a failure value.
- Whether any dependency mutates receiver fields, globals, input contents, or dispatch information.
- The identity and broader behavior of F-indirect-01 and F-unresolved-01.
- Whether a nonzero callback return means success, failure, or a request for another action.
- Valid concurrency, reentrancy, lifetime, alignment, and aliasing rules.
- Byte order and complete object layouts.
- Exceptional or nonlocal control transfers inside dependencies.

No rollback, error translation, ownership transfer, or resource cleanup should be inferred beyond the established call sequence.

## Evidence

This contract is based solely on static inspection of the supplied function representation and the supplied opaque dependency mapping.

Established evidence comprises the explicit field accesses, argument widths and constants, call order, signed comparisons, conditional store, dispatch offset, and normal return paths. Allocation-like and notification-like interpretations are inferred intent only.

No runtime tests, dependency implementations, memory traces, or executable observations have been provided or performed.
