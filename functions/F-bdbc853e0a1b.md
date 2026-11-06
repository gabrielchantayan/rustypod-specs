<!-- Trusted provenance: F-bdbc853e0a1b; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-bdbc853e0a1b

## Purpose

F-bdbc853e0a1b routes an event through an object’s dynamically selected handlers. It performs an optional notification, offers certain events to a preliminary handler, invokes one of two main handlers, and conditionally invokes a final handler.

A nonzero handler result stops processing and is returned unchanged. If no invoked handler returns a nonzero result, the function returns zero.

The routing decisions are made in stages. Event fields and the object’s dispatch-table pointer are read again after intervening calls. Consequently, mutations made by a dependency can affect later routing within the same invocation.

The interpretation of these calls as “notification” and “handlers” is inferred intent. Their selection conditions, arguments, ordering, and return propagation are established by the supplied static evidence.

## Interface

### Inputs

The function receives two arguments:

- **Object reference:** a reference to storage containing a dispatch-table pointer and additional object fields.
- **Event reference:** an address identifying an event record.

Neither argument is checked for nullness or validity.

The supplied representation uses integer-valued addresses and integer fields. The following layout describes the apparent 32-bit model: four-byte integer and pointer slots, with byte offsets measured from the corresponding base address. The original ABI and type declarations are not supplied; their exact guarantees remain documented_only.

### Object layout used by this function

| Byte offset | Observed use |
|---|---|
| `+0x00` | Pointer to the current dispatch table. Read separately when a dynamically dispatched call is reached. |
| `+0x50` | Integer-valued comparison field used by the final event-routing condition. |
| `+0x10C` | Integer-valued field converted to a character-width argument for F-4fd087f2325e. |

The object field at `+0x10C` and the dispatch-table entry at `+0x10C` are in different storage regions. They must not be conflated.

### Event and nested-record layout

| Storage | Byte offset | Observed use |
|---|---:|---|
| Event | `+0x04` | Event discriminator. Compared with `0x16` and, later, `0x0F`. |
| Event | `+0x08` | Nested-record address when the discriminator is `0x16`; direct comparison value when it is `0x0F`. |
| Nested record | `+0x04` | Secondary discriminator used for notification and handler selection. |

No event field at offset zero is read directly by this function.

The nested record is accessed only when the event discriminator observed at that stage equals `0x16`. A different discriminator does not require a valid nested-record address for the function’s own direct reads, although a called dependency may impose additional requirements.

### Dispatch-table entries

Each selected entry is invoked with the original object reference and original event reference.

| Dispatch-table byte offset | Role in this contract |
|---|---|
| `+0x124` | Preliminary handler for secondary discriminator zero. |
| `+0x10C` | Main handler selected for secondary discriminator `0x2A`. |
| `+0x110` | Main handler selected for every other main-routing case. |
| `+0x0E0` | Final handler for a matching `0x0F` event. |

These are indirect dependency slots, not independently identified functions. Their opaque function IDs are unavailable.

### Output

The function returns an integer:

- The exact nonzero result of the first preliminary, main, or final handler that returns nonzero.
- Zero when all reached handlers return zero.

The result of F-4fd087f2325e is not used to select the function’s return value.

### Preconditions

For a defined execution, every read and indirect call actually reached must be valid:

- The event discriminator must be readable at each stage that reaches it.
- A discriminator of `0x16` requires a readable nested-record address and secondary discriminator at that stage.
- The notification path requires the object field at `+0x10C` to be readable.
- Each indirect call requires a readable object dispatch-table pointer, a readable selected entry, and an appropriately callable target.
- The matching-final-event path requires the object field at `+0x50` to be readable.

These requirements apply to the state at the time of access, including after earlier dependencies execute. There is no local validation or recovery path for invalid storage or an invalid call target.

## Behavior

### 1. Optional notification

The function first reads the event discriminator.

If that value is `0x16`, it reads the event’s nested-record address and then the secondary discriminator at nested offset `+0x04`. It subtracts `0x2500` from that integer value and compares the result with `2`.

Under the apparent 32-bit arithmetic model, the matching secondary value is `0x2502`.

If the comparison matches, the function reads the object field at `+0x10C`, converts its value to the character-width type shown in the supplied representation, and invokes F-4fd087f2325e with:

1. The original object reference.
2. The converted field value.

The character conversion does not pass the full integer field unchanged. Its precise signedness and out-of-range conversion semantics are not established.

If the initial discriminator is not `0x16`, no nested-record read is performed during this stage. If the secondary value does not match, no notification call occurs.

There is no return-result gate after the notification. If the dependency returns normally, processing proceeds to the next stage.

### 2. Preliminary handling of secondary discriminator zero

The event discriminator is read again, even if it was already read during notification selection.

If its current value is `0x16`, the function freshly reads the nested-record address and its secondary discriminator. If that secondary value is zero, it obtains the current dispatch-table pointer from the object and invokes the entry at `+0x124`.

- A nonzero result is returned immediately.
- A zero result allows processing to continue.

If the discriminator is not `0x16`, or its secondary discriminator is not zero, this preliminary handler is not called.

The earlier notification call may have changed the event, nested record, object, or dispatch table. This stage uses the values it reads afterward; it does not reuse a saved notification-stage classification.

### 3. Main handler selection

Unless the preliminary handler has already returned a nonzero result, the function again reads the event discriminator.

If it is `0x16`, the function freshly reads the nested-record address and secondary discriminator:

- Secondary discriminator `0x2A` selects dispatch-table entry `+0x10C`.
- Every other secondary discriminator selects entry `+0x110`.

If the event discriminator is not `0x16`, entry `+0x110` is selected without reading a nested record for this decision.

The function obtains the current dispatch-table pointer from the object and calls the selected entry with the original two arguments.

- A nonzero result is returned immediately.
- A zero result allows the final stage to run.

Exactly one of these two main entries is called on any normally executing path that reaches this stage. The `+0x110` entry is not a fallback attempted after a zero result from `+0x10C`.

### 4. Conditional final handling

After a zero result from the main handler, the function reads the event discriminator again.

If that value is `0x0F`, it reads:

1. The event value at `+0x08`.
2. The object comparison field at `+0x50`.

If these values are equal, it obtains the object’s current dispatch-table pointer and invokes the entry at `+0x0E0`.

- A nonzero result is returned unchanged.
- A zero result leads to the function’s final zero return.

If the discriminator is not `0x0F`, neither comparison field is read by this stage. If it is `0x0F` but the values differ, the final handler is skipped.

The final condition uses state observed after the main handler returns. An event that was not initially `0x0F` can therefore qualify if an earlier dependency changes it.

### 5. Completion

When processing reaches the end without a nonzero handler result, the function returns zero.

An early return prevents all subsequent routing reads and calls. For example, a nonzero main-handler result prevents the final discriminator read, not merely the final call.

## Side effects

### Direct effects

No explicit write to the object, event, nested record, or dispatch table appears in the supplied function.

Its direct effects consist of:

- Reading fields and dispatch entries.
- Calling selected dependencies.
- Returning an integer result.

No allocation, deallocation, ownership transfer, reference-count update, logging, or explicit error reporting is visible in this function.

### Dependency-mediated effects

The dependencies’ effects are unknown. This contract does not assume that they:

- Preserve object or event contents.
- Preserve the object’s dispatch-table pointer.
- Preserve global state.
- Retain or release any particular resource.
- Leave the arguments’ storage alive.
- Return normally.

The original object and event references are reused in later stages. If a dependency invalidates storage needed by a later read, the function has no visible mechanism to detect or repair that condition.

The absence of a local allocation or release does not establish an ownership policy for the wider call sequence.

### Ordering

The stage order is fixed:

1. Optional F-4fd087f2325e notification.
2. Optional preliminary handler.
3. One main handler, if processing has not stopped.
4. Optional final handler, if the main handler returned zero.
5. Zero return if no handler produced a nonzero result.

Within each routing decision, the relevant event discriminator is read before any conditionally required nested record. For the final comparison, the event value is read before the object comparison field.

This is the access ordering exposed by the supplied representation. It is not evidence of atomicity, volatile semantics, synchronization, or a supported concurrent-mutation model.

## Dependencies

### F-4fd087f2325e

**Selection:** Initial-stage event discriminator `0x16`, with a nested secondary value satisfying the subtraction comparison described above.

**Arguments:** Original object reference and a character-width conversion of the object integer field at `+0x10C`.

**Return handling:** No result is used by this function.

**Unknowns:** Purpose, mutations, ownership behavior, global effects, and abnormal-exit behavior.

### Indirect dispatch dependencies

The four slots listed in the interface are selected independently at their respective stages. Each is passed the original object and event references.

Their integer results have the same local control meaning: zero continues processing, and nonzero stops processing and is propagated unchanged.

No additional behavior is established for these targets. In particular, a slot name in this contract describes its position in routing, not a guarantee about its implementation.

## Acceptance examples

These are conditional acceptance cases derived from static evidence. They are not executed tests. Unless a case explicitly includes mutation, assume valid storage, normally returning callees, and unchanged routing fields during the invocation.

### Secondary discriminator `0x2502`

For event discriminator `0x16` and secondary discriminator `0x2502`:

- F-4fd087f2325e is called with the object and converted field value.
- The preliminary `+0x124` entry is skipped.
- The main `+0x110` entry is called.
- If that handler returns zero and the event remains unchanged, the final handler is skipped and the result is zero.

This case assumes the apparent 32-bit subtraction interpretation.

### Preliminary handler terminates processing

For event discriminator `0x16` and secondary discriminator zero:

- The notification is skipped.
- Entry `+0x124` is called.
- If it returns `-7`, the function returns `-7`.
- Neither main entry nor the final entry is called.

Nonzero results are not normalized to a Boolean value.

### Preliminary handler permits processing

For the same event classification, if `+0x124` returns zero and leaves routing fields unchanged:

- Entry `+0x110` is called next.
- Its nonzero result, if any, becomes the function result.
- If it returns zero, the final stage observes discriminator `0x16` and returns zero without calling `+0x0E0`.

### Special main handler

For event discriminator `0x16` and secondary discriminator `0x2A`:

- The notification and preliminary handler are skipped.
- Entry `+0x10C` is called.
- Entry `+0x110` is not called, including when `+0x10C` returns zero.

### Matching final event

For event discriminator `0x0F`, with the event value at `+0x08` equal to the object field at `+0x50`:

- No nested-record dereference is required by this function.
- Entry `+0x110` is called first.
- If it returns zero, entry `+0x0E0` is called.
- If the final handler returns `11`, the function returns `11`.

If the main handler instead returns nonzero, the comparison and final call do not occur.

### Mutation changes later routing

Suppose the preliminary handler is reached for secondary discriminator zero, changes that discriminator to `0x2A`, and returns zero.

The main stage must use the newly read value and select `+0x10C`, rather than selecting from the original zero value.

Similarly, if a zero-returning main handler changes the event to discriminator `0x0F` and establishes matching comparison fields, the final handler is eligible even when the event had a different discriminator at entry.

### Irrelevant nested pointer

For an unchanged event whose discriminator is neither `0x16` nor `0x0F`:

- The function’s own routing reads do not dereference the event field at `+0x08`.
- Entry `+0x110` is called.
- A zero result from that entry leads to zero return.

This does not establish that the called handler ignores that field.

## Uncertainties

The following remain documented_only:

- Original ABI, calling convention, pointer width, integer-width guarantees, alignment requirements, and byte order. The layout above follows the apparent four-byte model.
- Character width, plain-character signedness, and conversion behavior for values outside its representable range.
- Whether subtraction has target-level wrapping behavior or source-language signed-overflow constraints. The supplied subtraction form alone does not establish a portable language-level rule for every integer input.
- Semantic meanings of the event constants and object fields.
- Concrete identities and implementations of the indirect call targets.
- Dependency mutations, global effects, memory ownership, resource lifetime, exceptions, and nonlocal exits.
- Concurrency support and whether any reads correspond to specially qualified storage.

There is no visible explicit error return for invalid arguments. That absence must not be interpreted as evidence that invalid arguments safely produce zero.

## Evidence

**Established static evidence:** The supplied representation exposes the staged field reads, nested-record conditions, object and dispatch-table offsets, dependency arguments, indirect-call selection, early returns, and final zero return described above. It contains no explicit memory writes or local resource-management operations.

**Inferred intent:** “Event,” “notification,” “preliminary,” “main,” and “final” are descriptive roles assigned to the observed routing structure. Zero plausibly means “continue” or “not handled,” but only its control-flow effect is established.

**Verification limit:** No runtime tests have been performed. The acceptance examples specify conditional behavior supported by the static evidence; they do not claim observed execution results.
