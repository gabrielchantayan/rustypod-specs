<!-- Trusted provenance: F-4beaec2f30ca; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-4beaec2f30ca

## Purpose

F-4beaec2f30ca conditionally performs a preliminary operation and, if that operation returns zero, invokes an indirect operation that can select an object for a second indirect invocation.

The function returns either a snapshot of a global 32-bit value or the result of the preliminary operation. Neither indirect invocation has a visible return-value role.

The established behavior is a gated dispatch sequence. The meaning of the gate, the global value, the preliminary result, and the two indirect operations is not established. In particular, interpreting zero as “success,” or the final invocation as destruction or release, would be inferred intent rather than a supported behavioral guarantee.

## Interface

### Inputs

The visible interface has four argument positions:

| Position | Visible representation | Established use |
|---|---|---|
| 1 | 32-bit integer used as an address | Identifies the base of a structure containing a byte at displacement `0xEC` and an object-pointer field at displacement `0xE8`. Also passed to F-c03d5430bb19. |
| 2 | Uninterpreted 32-bit value | No visible use. |
| 3 | Uninterpreted 32-bit value | No visible use. |
| 4 | Pointer to a 32-bit integer-sized location | Used as the initial value of a local pointer slot. It is not directly dereferenced on entry. |

The pointer-bearing operations are consistent with a 32-bit address model. The supplied representation does not independently establish the complete execution ABI, byte order, or alignment rules.

The second and third inputs do not affect any visible branch, read, write, invocation, or return-value selection. They are not forwarded by the visible operations.

### Output

The function returns a 32-bit integer value:

- If the byte at base displacement `0xEC` is zero, it returns the global value read at entry.
- If that byte is nonzero, it returns the value produced by F-c03d5430bb19, including when that value is zero.

A negative preliminary result is handled like any other nonzero result: it is returned and suppresses both indirect invocations.

### Relevant layouts

Only the following portions of the addressed structures are established:

| Structure or storage | Displacement | Meaning in this function |
|---|---:|---|
| Primary structure | `0xE8` / 232 bytes | Pointer to a dispatch object |
| Primary structure | `0xEC` / 236 bytes | One-byte gate, tested for zero versus nonzero |
| Dispatch object | `0x00` | Pointer to a dispatch table |
| First dispatch table | `0x0C` / 12 bytes | Callable entry for the first indirect dependency |
| Selected object | `0x00` | Pointer to a dispatch table |
| Second dispatch table | `0x08` / 8 bytes | Callable entry for the second indirect dependency |

The two dispatch tables need not be the same table. The two objects need not be the same object.

The displacements are byte offsets. No array length, complete object size, table length, type tag, or allocation boundary is supplied.

### Preconditions for normal completion

Every address actually traversed must be valid for its corresponding read or invocation. In particular:

- The entry global must be readable.
- The primary structure’s gate byte must be readable.
- If the first indirect invocation is reached, its object-pointer field, object header, table entry, and callable target must be usable.
- If the selected pointer is nonnull, its object header, table entry, and callable target must be usable.
- Invoked dependencies must return normally for this function to return normally.

The implementation exposes no checks enforcing these conditions. A nonnull pointer is not otherwise validated.

## Behavior

### Entry snapshot and gate

The first visible operation reads a global 32-bit value into the function’s return-value state. This read occurs even when later behavior replaces that value with the preliminary result.

The function then reads the single byte at primary-base displacement `0xEC`.

If that byte is zero:

- The preliminary dependency is not invoked.
- The dispatch-object field is not traversed by the visible body.
- Neither indirect dependency is invoked.
- The entry snapshot is returned.

The gate is a byte-level zero test. Every nonzero byte representation takes the enabled path; no particular Boolean encoding is required.

### Preliminary operation

When the gate byte is nonzero, the function initializes a local pointer slot with the fourth input and invokes F-c03d5430bb19 with the first input.

The preliminary result replaces the entry global snapshot as the return-value state.

If that result is nonzero, processing stops and that result is returned. The local pointer slot is not used for a subsequent indirect invocation on this path.

If that result is zero, the function proceeds to the first indirect invocation.

The gate byte is not visibly reread after F-c03d5430bb19. A change to that byte during the preliminary operation therefore does not itself cancel the already-selected continuation.

### First indirect invocation

The first indirect dependency is designated F-000000000001 in this document. This is an opaque, document-local label for an unresolved target, not a recovered function identity.

Its target is obtained through:

1. The dispatch-object pointer field at primary-base displacement `0xE8`.
2. The dispatch-table pointer at the beginning of that object.
3. The callable entry at table displacement `0x0C`.

The invocation has two visible arguments:

- An object pointer obtained from the primary structure’s dispatch-object field.
- The address of the local pointer slot initialized from the fourth input.

Consequently, the dependency can inspect or replace that slot. The visible body does not require it to replace the slot.

The dispatch-object field is consulted after F-c03d5430bb19 returns. Changes made to that field by the preliminary dependency can therefore affect dispatch.

The supplied expression contains separate uses of the dispatch-object field for target resolution and for the object argument. It does not establish a portable evaluation order between those uses or guarantee one physical load. Without stable storage, this contract does not promise that the target and object argument originate from a single consistent snapshot.

The first indirect dependency’s return value is not tested or used to select the function’s returned value.

### Selected-pointer check and second invocation

After the first indirect dependency returns, the function examines the current contents of the local pointer slot.

If the slot is null, the second indirect invocation is skipped.

If the slot is nonnull, the second indirect dependency is invoked. This dependency is designated F-000000000002, another document-local opaque label for an unresolved target.

Its target is obtained through:

1. The pointer currently held in the local slot.
2. The dispatch-table pointer at the beginning of the selected object.
3. The callable entry at table displacement `0x08`.

No explicit arguments are represented for this second invocation. In particular, the supplied representation does not explicitly pass the selected object as a receiver. An ABI-specific receiver or other implicit call context is not established and must not be invented.

If the first indirect dependency leaves the slot unchanged, the fourth input remains the selected pointer. A nonnull fourth input can therefore lead to the second invocation without any replacement being written.

### Return

On the path reaching either indirect dependency, the preliminary result was zero. If the dependencies return normally and do not invalidate the caller’s execution state, the function returns zero after the indirect processing.

There is no visible reread of the entry global before returning. Ordinary changes to that global by dependencies do not replace the already-selected return value.

## Side effects

### Directly visible effects

The function visibly:

- Reads the entry global.
- Reads the gate byte.
- Conditionally reads object pointers, dispatch-table pointers, and callable entries.
- Initializes a local pointer slot on the enabled path.
- Exposes the address of that slot to F-000000000001.
- Conditionally invokes three dependencies in a fixed high-level order.

It contains no explicit write to the primary structure, the entry global, or either selected object. This is not a guarantee that those locations remain unchanged: invoked dependencies may modify accessible state.

### Ownership and lifetime

No allocation, ownership transfer, reference-count adjustment, or release operation is established directly.

The final indirect invocation must not be described as a release merely because it occupies a particular table displacement. Its resource effects are unknown.

The local pointer slot is temporary storage belonging to this invocation. Passing its address establishes neither permission to retain it after return nor a valid lifetime for later asynchronous access.

The fourth input’s pointee is not directly read or written through an integer-data access by this function. If that pointer remains selected, however, the same address is interpreted as an object whose first word supplies a dispatch-table pointer.

### Errors and abnormal completion

There is no explicit error construction, retry, fallback dispatch, exception handler, or cleanup path.

A nonzero preliminary result is returned without further interpretation. A null selected pointer suppresses the final invocation rather than causing a visible error return.

Null dispatch-object pointers, invalid table pointers, unusable callable entries, and invalid nonnull selected pointers are not guarded. Their actual consequences depend on the runtime and dependency behavior; the supplied evidence does not establish a specific fault or recovery mechanism.

## Dependencies

| Opaque function ID | Invocation condition | Visible interface and result use |
|---|---|---|
| F-c03d5430bb19 | Gate byte is nonzero | Receives the first input. Its 32-bit result becomes the function’s return value and permits further dispatch only when zero. |
| F-000000000001 | Gate is nonzero and preliminary result is zero | Indirect target at first dispatch-table displacement `0x0C`. Receives an object pointer and the address of the local pointer slot. Its return value is unused. |
| F-000000000002 | First indirect dependency returns with a nonnull selected pointer | Indirect target at selected object’s table displacement `0x08`. No explicit arguments are represented. Its return value is unused. |

The last two IDs identify dispatch roles only. Their concrete target identities may vary with object state.

No dependency is assumed to preserve globals, preserve object fields, own memory, or release memory.

## Acceptance examples

These are conditional acceptance cases derived from static evidence, not reported executions. All assume valid memory, compatible call targets, normal dependency return, and no interference that corrupts execution state.

1. **Disabled gate.**  
   If the entry global contains 37 and the gate byte is zero, the result is 37. No dependency is invoked. The fourth input may be null without triggering a dereference.

2. **Enabled gate with a nonzero preliminary result.**  
   If the gate is nonzero and F-c03d5430bb19 returns −4, the function returns −4. Neither indirect dependency is invoked, regardless of the fourth input.

3. **Entry global does not determine the enabled result.**  
   If the entry global contains 91, the gate is nonzero, and the preliminary result is zero, the eventual normal return is zero. This remains true if an indirect dependency changes the global to another value.

4. **Null initial selection remains null.**  
   If the fourth input is null and F-000000000001 leaves the slot null, F-000000000002 is skipped.

5. **An unchanged nonnull selection is used.**  
   If the fourth input identifies a valid selected object and F-000000000001 leaves the slot unchanged, the function invokes the callable entry at that object’s table displacement `0x08`.

6. **Replacement selection is used.**  
   If F-000000000001 replaces the slot with a different valid object pointer, the second dispatch uses the replacement rather than the original fourth input.

7. **Clearing the selection suppresses dispatch.**  
   If the fourth input is nonnull but F-000000000001 writes null into the slot, there is no second invocation.

8. **The preliminary dependency changes the dispatch object.**  
   If F-c03d5430bb19 installs a different valid dispatch-object pointer and returns zero, subsequent dispatch consults the post-call field rather than a pointer explicitly cached before the call.

9. **The preliminary dependency clears the gate.**  
   If the gate was initially nonzero, the preliminary dependency returns zero, and it clears the gate, the function still proceeds to the first indirect invocation. There is no second visible gate test.

10. **Indirect result values do not become the return value.**  
    If either indirect dependency produces a machine-level return value, that value does not replace the preliminary zero in the visible behavior.

## Uncertainties

The following items are **documented_only**:

- The semantic purpose, address, initialization rules, and synchronization of the entry global.
- The complete primary-structure and object layouts.
- The complete ABI, byte order, pointer alignment, and implicit argument conventions.
- The concrete identities and signatures of the indirect targets.
- Whether zero denotes success for F-c03d5430bb19.
- Whether either indirect dependency allocates, releases, retains, or transfers resources.
- Whether dependencies mutate globals, the primary structure, or selected objects.
- Whether the first indirect dependency retains the local slot’s address.
- Behavior under concurrent mutation, reentrancy, invalid pointers, incompatible targets, or abnormal dependency termination.
- Exact machine-load counts and argument-evaluation ordering within the first indirect-call expression.

Address calculations use fixed byte displacements. No bounded indexing or length arithmetic is visible. The evidence does not establish defined behavior for an address calculation that wraps the machine address space or reaches outside valid storage.

## Evidence

Established static evidence consists of the supplied function representation and the supplied opaque mapping for its direct dependency.

That evidence supports the entry-global snapshot, byte gate, preliminary-result replacement, short-circuit ordering, indirect table displacements, mutable local pointer slot, null selection check, and unused indirect return values.

The representation supports a 32-bit interpretation of the integer and pointer-bearing accesses, but it is not a complete ABI specification. No source-level type declarations, dependency bodies, runtime traces, or ownership contracts were supplied.

No runtime tests have been performed. The acceptance examples state required behavior under their listed preconditions; they are not execution results.
