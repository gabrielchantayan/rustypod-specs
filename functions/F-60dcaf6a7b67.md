<!-- Trusted provenance: F-60dcaf6a7b67; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-60dcaf6a7b67

## Purpose

F-60dcaf6a7b67 recognizes two selector values and, for either recognized value, issues an ordered pair of indirect calls through an acquired object. Each call receives the same object and the same caller-supplied four-byte payload, but a different selector-dependent value obtained from global storage.

A normally completed invocation returns a logical success indication for either recognized selector and a logical failure indication for every other selector. This indication reflects selector recognition and completion of the visible control flow; it does not establish that either indirect operation succeeded.

The application-level meaning of the two selectors, the four global values, and the indirect operation is not established. Describing the operation as registration, notification, configuration, or resource management would exceed the available evidence.

## Interface

The recovered interface has three positional inputs and a Boolean result:

| Position | Input | Established treatment |
|---|---|---|
| First | Four-byte value | Not read or forwarded by the visible function body. |
| Second | Integer selector | Compared with `0x8C81` and `0x8C82` using exact equality. |
| Third | Four-byte payload | Forwarded unchanged as the third argument of both indirect calls on a recognized path. |

“Four-byte” specifies a 32-bit value in the recovered representation. It does not establish whether that value is an integer, pointer, handle, packed field, or another application-specific quantity.

The selector is represented as an integer in the supplied evidence. The excerpt does not independently establish the target integer ABI. Both recognized constants are positive, fit in a signed 32-bit integer, and require no overflow, truncation, sign extension, or arithmetic transformation for comparison. Under a 32-bit selector interpretation, the recognized bit patterns are `0x00008C81` and `0x00008C82`.

The result is logically:

- **True:** the selector is recognized and both indirect calls return to this function.
- **False:** the selector is unrecognized and the acquisition dependency returns to this function.

The machine-level Boolean return encoding, register assignment, stack layout, and calling convention are not established.

The first input is unused by the visible body. This does not eliminate any calling-convention requirement to supply that argument.

### Operational preconditions

Every invocation reaches the object-acquisition dependency before selector-specific processing. Consequently, normal completion for any selector requires that dependency to return.

For a recognized selector, normal completion additionally requires:

- The returned object supports the initial pointer dereference.
- The pointer stored at the beginning of that object identifies storage from which a callable target can be read at byte offset `0x54`.
- The target accepts the observed three-argument call shape.
- After the first indirect call returns, the original object address remains usable for another dispatch lookup.
- The global storage used by the selected branch remains readable when accessed.

These are operational requirements, not validations performed by F-60dcaf6a7b67. No explicit null check, lifetime check, or dispatch-target validation is visible.

## Behavior

### Common entry behavior

The first observable dependency invocation is F-9d0ecc5f6adf. It is called without visible arguments. Its result is retained as the object address used by subsequent dispatches.

This acquisition occurs even when the selector is unsupported. The function does not first reject unsupported selectors to avoid acquisition.

After acquisition returns, the selector determines one of three paths:

| Selector | First dispatched value | Second dispatched value | Normal result |
|---|---|---|---|
| `0x8C81` | Global value A | Global value B | True |
| `0x8C82` | Global value C | Global value D | True |
| Any other value | None | None | False |

A, B, C, and D are descriptive labels for four distinct global storage locations referenced by the supplied evidence. They are not asserted to have distinct stored values.

### Unsupported selectors

For every selector other than the two recognized constants:

1. Object acquisition has already occurred.
2. No indirect dispatch is performed by this function.
3. The function returns false.

No visible dereference of the acquired object is required on this path. An unusable acquisition result therefore does not, by itself, cause a visible dereference within this function when the selector is unsupported. The acquisition dependency may nevertheless fail or have effects of its own.

There is no fallback dispatch, range-based recognition, or treatment of nearby selector values as equivalent.

### Recognized selectors

For either recognized selector, the function performs two calls in sequence.

The first call uses a target obtained through the acquired object:

- Read the pointer stored at the beginning of the object.
- Resolve a callable target at byte offset `0x54` from that pointer.
- Obtain the branch’s first global value.
- Invoke the target with the acquired object, that global value, and the unchanged four-byte payload.

The selected first global value is A for `0x8C81` and C for `0x8C82`.

The source establishes that the first call must return before the second dispatch lookup and second global-value access take place. It does not establish a portable machine-instruction ordering among all reads needed to prepare the first call.

After the first call returns, the function performs a fresh dispatch lookup through the same retained object address. It does not simply reuse the target resolved for the first call.

The second-call preparation then obtains the branch’s second global value, B or D, and invokes the newly resolved target with:

- The same retained object address as the first argument.
- The second global value as the second argument.
- The same caller-supplied four-byte payload as the third argument.

The visible sequence resolves the second target before obtaining the second global value. Exact instruction-level memory ordering is not independently established.

If the second call returns, the function returns true.

### Effects of changes during the first call

The fresh lookup is behaviorally significant. If the first call changes the dispatch pointer stored in the object, or changes the applicable dispatch entry, the second call can use the changed target.

Likewise, the second global value is obtained after the first call returns. If the first call changes that global storage, the second call can receive the updated value rather than its entry-time value.

No preservation guarantee for object contents or global state may be imposed on the indirect target based on this excerpt. Conversely, the excerpt does not prove that such mutations actually occur.

The original object address is retained for the second call. A first-call return value is not used to select a replacement object.

### Completion and failure behavior

Neither indirect call’s return value is inspected. The function exposes no branch based on those results and performs no visible error-code translation.

Accordingly, a callee that reports failure through an ordinary return value does not cause F-60dcaf6a7b67 to return false. If both calls return normally, a recognized selector produces true.

There is no visible local recovery, retry, rollback, compensating operation, or cleanup path. If acquisition or an indirect call does not return normally, the excerpt provides no basis for promising a Boolean result. The precise consequences of an invalid access or incompatible call target depend on the execution environment and remain undocumented here.

## Side effects

The visible body contains no direct assignment to caller-provided storage, object fields, or global data.

Its externally relevant effects include:

- One acquisition-dependency invocation on every entered path.
- For recognized selectors, reads supporting two indirect dispatches.
- For recognized selectors, reads of the corresponding two global values.
- Two ordered indirect calls for a recognized selector, provided the first returns.
- Any effects produced transitively by those calls.

The payload is forwarded as a value. The function does not visibly dereference it. If it represents an address or handle, any interpretation, retention, mutation, or release by the callees is unknown.

The function visibly performs no allocation or release. This does not establish that acquisition allocates nothing, that the acquired object is borrowed, or that either indirect call leaves ownership unchanged. Ownership and lifetime contracts are not recoverable from the available evidence.

No synchronization is visible. The sequential observations above describe the function’s own control flow and reads around returning calls; they do not establish thread safety or guarantees for concurrent mutations.

## Dependencies

### F-9d0ecc5f6adf

This is the identified direct dependency.

Established contract at this call site:

- Invoked once before selector handling.
- Receives no visible arguments.
- Supplies the object address used by recognized paths.
- Is invoked even when no indirect calls will follow.

Unknowns include its return type beyond this usage, failure signaling, allocation behavior, ownership policy, global side effects, synchronization, and whether it always returns the same object.

### Unresolved indirect dependency — documented_only

The callable dependency is selected through a dispatch entry at byte offset `0x54`. No opaque function ID is supplied for the possible targets, so their identities remain unresolved.

The observed call shape consists of three arguments: object address, selected four-byte global value, and caller-supplied four-byte payload. Return values are ignored.

The two invocations need not resolve to the same target. Neither identical targets nor different targets are required: the result depends on the dispatch state observed at each lookup.

### Global data dependencies — documented_only

Four global storage locations provide the selector-dependent values A through D. Their contents, initialization rules, mutability constraints, application meanings, and relationships are unknown.

The offset `0x54` is a byte displacement, equivalent to 84 decimal bytes. It must not be converted into a fixed dispatch-slot index without establishing the target’s pointer width and table layout.

## Acceptance examples

These examples are conditional acceptance criteria derived from static evidence. They are not reports of executed tests.

1. **Recognized first selector:** Given selector `0x8C81`, successful acquisition, valid dispatch state, and normally returning callees, observe acquisition followed by calls carrying A and then B. Both receive the same acquired object address and payload. The final result is true.

2. **Recognized second selector:** Under equivalent conditions for `0x8C82`, observe C and then D, with a final result of true. A and B are not selected by this branch.

3. **Unsupported selector:** Given selector `0x8C80`, if acquisition returns, observe no indirect calls and a false result. Acquisition must still occur.

4. **Unused first input:** If only the first input changes and dependency behavior is otherwise held fixed, the visible body makes the same selector choice and forwards the same payload. No visible call receives the first input.

5. **Changed dispatch after the first call:** If the first indirect call replaces the applicable dispatch target while preserving valid object access, the second call uses the target obtained by the fresh lookup.

6. **Changed second global value:** If the first call updates B on the `0x8C81` path, the second call receives the value subsequently read from B, not a value required to have been captured at entry. The corresponding condition applies to D on the other recognized path.

7. **Failure reported by ordinary return:** If both indirect calls return normally but return values conventionally indicating failure, F-60dcaf6a7b67 still returns true for a recognized selector.

8. **Nonreturning first call:** If the first indirect call never returns, the second call and final true result are not reached.

## Uncertainties

**Inferred intent:** The two recognized selectors appear to request paired operations on an acquired object. Their domain-specific purpose is not established.

**documented_only:**

- Exact target ABI, pointer width, integer width, and Boolean encoding.
- Valid payload meanings and values.
- Meanings and initial contents of A, B, C, and D.
- Acquisition failure semantics and object ownership.
- Indirect-target identities and their memory-management contracts.
- Whether callees modify dispatch state or global storage in practice.
- Exception, fault, and nonlocal-exit behavior.
- Concurrency guarantees and instruction-level ordering beyond the visible sequential dependencies.

These unknowns must not be replaced with assumed preservation, ownership, validation, or success guarantees.

## Evidence

The established behavior comes solely from static inspection of the supplied recovered function body and the supplied opaque dependency mapping.

The visible evidence supports unconditional acquisition, exact selector matching, two ordered dispatches on recognized paths, a fresh second dispatch lookup, branch-specific global reads, unchanged payload forwarding, ignored indirect return values, and the stated Boolean outcomes.

No runtime tests have been performed. No external implementation, ABI documentation, global initializers, or callee bodies were available for this contract.
