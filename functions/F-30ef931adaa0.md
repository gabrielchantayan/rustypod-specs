<!-- Trusted provenance: F-30ef931adaa0; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-30ef931adaa0

## Purpose

F-30ef931adaa0 conditionally invokes F-90ffec85402c on a handle stored in a caller-supplied context. Its decision depends on a requested 32-bit value and information obtained through an associated object’s dispatch table.

Two decision routes are visible:

- When an initial temporary object produces a zero classification, the function compares the requested value with a value returned through dispatch slot +0x158. If those values differ, it compares scalar results extracted from two additional temporary objects.
- When the initial classification is nonzero, the function derives a value using the initial temporary object and a result from dispatch slot +0x17c, then compares that derived value with the request.

F-90ffec85402c is invoked when the applicable comparison permits it. The source does not establish what that invocation means at the application level. Descriptions such as “selection,” “activation,” “notification,” or “release” would be inferred intent, not established behavior.

## Interface

### Inputs

The function accepts three machine-level arguments:

| Position | Visible representation | Established use |
|---|---|---|
| First | 32-bit address-bearing value | Identifies a context whose field at byte offset +4 is read repeatedly. |
| Second | Opaque 32-bit value | Not referenced by the visible body and not forwarded to any visible dependency. |
| Third | 32-bit integer value | Requested comparison value; also passed to dispatch slot +0x170 on one route. |

The function has no visible return value. It does not expose its branch decision through a returned status.

### Context and dispatch layout

The visible access pattern establishes only the following layout:

- Context byte offset +4 contains a 32-bit associated-object address or handle.
- A nonzero associated-object address is treated as pointing to an object whose first 32-bit word identifies a dispatch table.
- Callable entries are read at dispatch-table byte offsets +0x158, +0x15c, +0x170, and +0x17c.

These offsets are byte offsets, not entry indices. No other context or object fields are established.

The function uses three distinct 24-byte temporary storage regions. Their internal representation, alignment requirements beyond those supplied by the original execution environment, and semantic type are not established.

### Preconditions

For a normally completing call:

- The first argument must permit reading the complete 32-bit field at context offset +4. A null first argument is not guarded.
- A nonzero associated-object value must support the object and dispatch-table reads required by the route taken.
- Each invoked dispatch entry must be a valid callable target with a compatible calling convention.
- Dependencies receiving temporary storage must accept the supplied storage and the visible argument forms.
- Dependency side effects must leave any subsequently accessed context, object, dispatch table, and temporary storage valid.

The initial nonzero check does not establish continued validity after dependency calls. The function does not lock, retain, or otherwise visibly stabilize the associated object.

No range restriction on the third argument is visible. Negative values and other 32-bit patterns are not rejected locally.

## Behavior

### Initial guard and temporary preparation

The first observable data access is a 32-bit read of the context field at offset +4.

If that value is zero, the function returns immediately. It does not initialize temporary storage, invoke a dispatch entry, or call any named dependency.

Otherwise, it initializes the first 24-byte temporary region by invoking F-ba5a4c1e0746 with that region and two zero-valued arguments. This establishes an initialization call, not proof that every byte becomes zero.

After initialization returns, the function rereads the associated-object field. Through that object’s dispatch table, it invokes slot +0x15c with two explicit arguments:

1. The associated object.
2. The first temporary region.

It then calls F-a7ff1a6124bd with the first temporary region. A zero return selects the first route below; any nonzero return selects the second.

The contents of the first temporary are opaque. The visible body does not inspect its bytes directly.

### Zero-classification route

The function rereads the associated-object field and invokes dispatch slot +0x158 through that object’s table. The visible call supplies no explicit arguments.

Its 32-bit result is compared with the third argument.

If they are equal, no additional temporary regions are initialized. The function proceeds to the final action.

If they differ, the function performs a second-stage comparison:

1. Initialize a second 24-byte temporary region through F-ba5a4c1e0746, supplying two zero-valued arguments.
2. Initialize a third 24-byte temporary region in the same way.
3. Reread the associated-object field and retain that particular object address for the next two dispatch calls.
4. Invoke slot +0x158 through the retained object’s dispatch table, again with no explicit arguments.
5. Invoke slot +0x170 through that retained object’s dispatch table, explicitly passing the retained object, the result just obtained, and the second temporary region.
6. Reread the associated-object field. Through the newly read object’s table, invoke slot +0x170 with that object, the requested third argument, and the third temporary region.
7. Call F-a43850bcdf96 with the second temporary region and retain its 32-bit result.
8. Call F-a43850bcdf96 with the third temporary region and retain its 32-bit result.
9. Call F-4e292a3dd745 on the third temporary region.
10. Call F-4e292a3dd745 on the second temporary region.
11. Compare the two retained extraction results.

If the extraction results differ, the final action is skipped. If they are equal, the final action is performed.

The second call to slot +0x158 is a fresh call. Its result is not required to equal the result that caused entry into this second-stage comparison. The function neither caches nor revalidates the earlier result.

Likewise, the two slot +0x170 calls need not use the same associated object: the second call obtains its receiver from a later context-field read.

Within the retained-object pair, retaining the object address does not freeze the dispatch table. The slot +0x170 target is obtained from that object after slot +0x158 returns.

### Nonzero-classification route

The function rereads the associated-object field and invokes dispatch slot +0x17c through its table. The visible call supplies no explicit arguments.

It passes the resulting opaque 32-bit value, together with the first temporary region, to F-45aa906a488b.

The 32-bit result from F-45aa906a488b is compared with the third argument:

- Equality permits the final action.
- Inequality skips the final action.

The second and third temporary regions are not initialized or cleaned up on this route.

### Final action and common cleanup

Whenever the applicable route permits the final action, the function rereads the context field at offset +4 and passes that 32-bit value to F-90ffec85402c.

This is not necessarily the object used for any earlier dispatch call. There is no fresh nonzero check before the invocation. If prior dependencies changed the field to zero, zero is the argument passed to F-90ffec85402c.

After either performing or skipping the final action, the function calls F-4e292a3dd745 on the first temporary region and returns.

Thus, on ordinary completion:

- A null initial associated-object value causes no cleanup call.
- A nonnull initial associated-object value causes cleanup of the first temporary exactly once.
- The two additional temporary regions are cleaned up only when the second-stage comparison was entered.
- Their cleanup order is third temporary, then second temporary.
- When those additional regions exist, both cleanup calls precede the final action.
- The first temporary’s cleanup follows the final action, if any.

These ordering statements assume dependencies return normally. No visible exception handler or nonlocal-exit recovery path establishes cleanup after an interrupted dependency call.

## Side effects

The function directly reads the context field and object dispatch structures. It supplies writable-looking addresses of temporary storage to dependencies, but no direct assignment to caller-owned storage is visible in the body.

This does not establish that the overall call is read-only. Any invoked dependency may have externally visible effects, including changes to the context field, object state, dispatch structures, globals, or temporary contents.

Repeated context reads are significant:

- Initialization can occur before the associated object used by slot +0x15c is obtained.
- Classification can occur before the object used by the selected route is obtained.
- Additional initialization can occur before the object retained for the second +0x158 call is obtained.
- The first +0x170 call can occur before the object for the second +0x170 call is obtained.
- Extraction and temporary cleanup can occur before the final-action argument is obtained.

No object ownership transfer, reference-count change, heap allocation, or memory release is established by this body alone. The cleanup-shaped calls must not be interpreted as proof that F-4e292a3dd745 frees memory. Similarly, passing the associated-object handle to F-90ffec85402c does not establish whether that dependency borrows, consumes, mutates, or releases it.

No explicit error result, logging, retry, bounds check, or fallback is visible. Invalid memory, invalid dispatch targets, or dependency failures have no locally specified recovery behavior.

## Dependencies

### Identified dependencies

| Opaque ID | Visible role and argument contract |
|---|---|
| F-ba5a4c1e0746 | Receives one 24-byte temporary region and two zero-valued arguments. Called once on every nonnull-entry route, and twice more on the second-stage comparison route. Return value is not used. |
| F-a7ff1a6124bd | Receives the first temporary region. Its 32-bit result selects zero versus nonzero classification. |
| F-a43850bcdf96 | Receives one additional temporary region. Called on the second region before the third; returned 32-bit values are compared for equality. |
| F-45aa906a488b | Receives the first temporary region and the opaque 32-bit result from slot +0x17c. Its 32-bit result is compared with the request. |
| F-90ffec85402c | Receives a fresh 32-bit read of the context’s associated-object field when the selected comparison permits the final action. Return value is not used. |
| F-4e292a3dd745 | Receives temporary storage at the cleanup points described above. Return value is not used. |

### Dynamically selected dependencies

The indirect targets have no supplied opaque function IDs. Their identities are therefore documented only by dispatch-table location:

- **Slot +0x15c:** explicitly receives the associated object and the first temporary region.
- **Slot +0x158:** returns a 32-bit comparison or descriptor-input value; no explicit arguments appear in the visible calls.
- **Slot +0x170:** explicitly receives an associated object, a 32-bit value, and one additional temporary region.
- **Slot +0x17c:** returns an opaque 32-bit value used by F-45aa906a488b; no explicit arguments appear in the visible call.

The absence of explicit arguments at +0x158 and +0x17c does not resolve hidden calling-convention inputs or justify adding an explicit receiver to their contract.

## Acceptance examples

These are conditional static acceptance examples, not executed tests. Dependency returns and state changes are assumed as stated.

### Initially absent associated object

Given a readable context whose offset +4 field is zero:

- The function returns.
- No dependency or dispatch call occurs.
- The second and third arguments do not affect this outcome.

### Direct equality in the zero-classification route

Given an initially nonzero associated object, a zero result from F-a7ff1a6124bd, and a first +0x158 result equal to the requested value:

- Only the first temporary is initialized.
- Neither +0x170 nor F-a43850bcdf96 is invoked.
- F-90ffec85402c receives the context field’s value as read at the final-action point.
- Cleanup of the first temporary occurs afterward.

### Different values, matching extracted results

Given zero classification, an initial +0x158 result different from the request, and equal results from the two F-a43850bcdf96 calls:

- Both additional temporaries are initialized.
- The first +0x170 call receives the result of the second +0x158 call, not the first.
- The second +0x170 call receives the requested value.
- The third and second temporaries are cleaned up, in that order.
- The final action occurs, followed by cleanup of the first temporary.

### Different extracted results

Under the same route, if the extraction results differ:

- Both additional temporaries are still cleaned up.
- F-90ffec85402c is not called.
- The first temporary is then cleaned up.

### Nonzero classification

Given a nonzero classification result:

- Slot +0x158 and slot +0x170 are not used.
- Slot +0x17c supplies the second argument to F-45aa906a488b.
- Equality between the derived result and the request permits the final action; inequality suppresses it.
- The first temporary is cleaned up in either case.

### Mutation before the final action

If an earlier dependency changes the context field, and the selected comparison permits the final action, F-90ffec85402c receives the changed field value. This includes zero. Acceptance of that value by F-90ffec85402c is outside the established contract.

## Uncertainties

The following are **documented_only**:

- The semantic meaning of the context, requested value, temporary objects, and dispatch results.
- The identities and full contracts of the indirect dispatch targets.
- Whether any dependency changes global state, context state, dispatch tables, or ownership.
- Whether initialization constructs resources and whether cleanup releases them.
- Signedness interpretations beyond zero/nonzero and equality comparisons. No visible ordering comparison or arithmetic depends on signedness.
- Calling-convention details not represented by explicit arguments.
- Behavior under exceptions, nonlocal exits, concurrency, reentrancy, invalid pointers, or dependency failure.
- Address wraparound behavior outside the valid-address preconditions.

There is no visible arithmetic on comparison values: they are passed through and compared at 32-bit width. No narrowing, widening, index scaling, or overflow-dependent numeric transformation is established.

## Evidence

The established behavior derives solely from the supplied static function body: its field offsets, 24-byte temporary regions, dependency arguments, conditional branches, repeated reads, equality comparisons, and cleanup order.

The supplied dependency map provides the opaque IDs used here. It does not establish dependency implementations or semantics.

“Initialization,” “classification,” “extraction,” “cleanup,” and “final action” are behavioral labels for visible call positions and result use, not assertions about hidden implementations.

No runtime tests have been performed. The acceptance examples express consequences of the visible control flow under stated dependency behavior; they are not observations of an executing program.
