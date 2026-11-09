<!-- Trusted provenance: F-0e0515d1ceed; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-0e0515d1ceed

## Purpose

F-0e0515d1ceed conditionally updates an object using a caller-supplied handle slot and a 32-bit option value.

Its visible normal result is:

- **Zero** when an entry gate skips processing, two captured handle words compare equal, an interrogation method returns zero, or preparation returns zero.
- **One** when the construction, destination-handling, and successful-path cleanup sequence completes normally.

Zero does not mean that the object was unchanged. Unsuccessful main-body paths can already have called dependencies, stored the option byte, assigned handle storage, and stored an interrogation result.

One does not establish that construction produced a nonzero handle or that a usable resource remains in the destination. Those conclusions require unavailable dependency and ownership contracts.

“Handle,” “interrogation,” “preparation,” “construction,” and “cleanup” describe apparent roles. They do not establish resource types or callee semantics.

## Interface

### Arguments and return value

| Input | Visible representation | Established use |
|---|---|---|
| Object | 32-bit address-like value | Base for field accesses and argument to dependencies. |
| Input handle slot | Address of a 32-bit word | Passed to an assignment dependency and read directly later. |
| Option | 32-bit value | Low byte stored in the object; full value passed to an indirect method. |

The normal return is a 32-bit value containing zero or one. No additional local error result is visible.

There is no local argument validation. Every reached memory access requires appropriate valid storage. Indirect calls require a valid receiver, dispatch table, and selected entry. Dependencies may impose additional prerequisites.

### Object layout observations

Offsets are byte offsets from the object base. This is a partial layout, not a complete object definition.

| Offset | Access width | Observed role |
|---|---:|---|
| `0x2B0` | 32 bits | Destination handle slot. |
| `0x2B8` | 32 bits | Stored interrogation result. |
| `0x2BC` | 8 bits | Entry gate. |
| `0x2C0` | 32 bits | Signed preparation bound. |
| `0x2F0` | 8 bits | Stored low byte of the option. |
| `0x2F4` | First word: 32 bits | Handle storage passed by address and compared directly. |

### Other storage

The function also uses:

- A global 32-bit word, observed twice on the nonzero-context path.
- An eight-byte local descriptor comprising two consecutive 32-bit words.
- Separate local input and output handle slots, each one 32-bit word.
- A four-byte temporary supplied to the comparison-handle dependency.
- A 32-bit field at handle referent offset `+4`, incremented on certain paths.
- A 32-bit field at output-related referent offset `+8`, tested before and after an increment.

These widths do not establish alignment requirements, complete referent layouts, or ownership.

## Behavior

### Entry and gate

F-b7cd573c0b4c is called without explicit arguments before the gate is read. The default result is zero.

The function then reads the byte at object offset `0x2BC`. If it is nonzero, the main body is skipped. F-d49b39475920 receives the object, and normal completion returns zero.

The controlling gate value is the observation after F-b7cd573c0b4c returns. The entry dependency is not established to preserve the gate or other state.

The skipped main body performs no direct option store, initial handle assignment, or indirect-method call. This does not exclude effects of the entry and common-exit dependencies.

### Initial setup and captured-word comparison

If the gate observation is zero, the following operations occur in order:

1. F-b7522c184a8b receives the object.
2. The option’s low eight bits are written at object offset `0x2F0`.
3. F-e4b7a6dcc8e5 receives the address of object storage at `0x2F4` and the caller’s input-slot address.
4. F-2e1e5059bccf receives the four-byte temporary’s address and zero.
5. A 32-bit word is read through the pointer returned by that dependency.
6. The first 32-bit word at object offset `0x2F4` is read.
7. F-fe0481c27e53 receives the temporary’s address.
8. The two previously captured words are compared.

The comparison therefore uses observations made before temporary cleanup, even if cleanup changes related storage.

If the words are equal, F-b7522c184a8b receives the object again. The function then takes the common zero-result exit.

Equality establishes only equality of the captured 32-bit representations. It does not establish resource identity. The zero argument to F-2e1e5059bccf does not prove that its returned word is zero or denotes an empty handle.

### Interrogation and preparation

If the captured words differ, F-56a66efac853 receives the address of object handle storage at `0x2F4`. Its returned pointer supplies a receiver and dispatch table. The entry at dispatch-table byte offset `+8` is invoked with that receiver and the full 32-bit option.

The method’s 32-bit result is written to object offset `0x2B8`.

A zero result causes another call to F-b7522c184a8b with the object, followed by the common zero-result exit.

For a nonzero result, the preparation argument is selected using signed 32-bit comparison:

- The method result is retained as one candidate.
- Object field `0x2C0` is read and retained as the other candidate.
- The bound field is read again for comparison against the method result.
- If the second bound observation is strictly less than the method result, the first bound observation is selected.
- Otherwise, the method result is selected.

With stable bound storage, this selects the signed minimum. Negative values are not rejected or clamped. With differing bound observations, the comparison uses the second observation, but selection of the bound supplies the first.

F-dcc406073ef9 receives the object and selected 32-bit value. Zero causes the second F-b7522c184a8b call and common zero-result exit. Any nonzero result permits construction processing.

### Context and descriptor initialization

After preparation succeeds, F-34ab68c7ba90 supplies a 32-bit construction-context value.

The function next:

1. Reads the global 32-bit word and retains this first observation for later descriptor cleanup.
2. Tests the context for zero.
3. If zero, calls F-a817d8d58c10, marked nonreturning in the supplied representation.
4. Otherwise, reads the global word again and initializes the descriptor’s first word from this second observation.
5. Initializes the descriptor’s second word to zero.

The first global read occurs even when the context is zero. The second read and descriptor initialization occur only on the nonzero-context path.

No dependency call intervenes between the two global reads on that path. Nevertheless, equality, immutability, and synchronization are not established. The first observation and the initialization observation remain distinct contract values.

On the zero-context path, no normal return or common-exit call is established. The mechanism used by F-a817d8d58c10 is unknown.

### Descriptor population and late input capture

With nonzero context, F-56a66efac853 resolves object handle storage at `0x2F4` again. The resulting receiver’s dispatch-table entry at byte offset `+0x30` is called with the receiver and descriptor address.

This is a fresh resolution. The descriptor is writable; the method’s visible return value is ignored.

Only after that method returns does the function read the caller’s input-slot word into the local input slot. If the captured word is nonzero, the referent’s 32-bit field at `+4` is read, incremented by one, and written back.

The late capture need not equal the value available to the earlier assignment dependency. No preceding dependency is established to preserve the caller’s slot.

F-962cf82b6591 then receives, in order:

1. The local output-slot address.
2. The retained nonzero context.
3. The descriptor address.
4. The local input-slot address.

The output slot has no visible initialization before this call. Normal continuation requires a usable output word to be established before subsequent reads. The dependency’s ordinary return value is not tested.

### Destination replacement and output-slot rereads

The address of object field `0x2B0` is compared with the local output-slot address.

If equal, the entire replacement branch is skipped: no destination cleanup, copy, output nonzero test, or output-related increment occurs. Successful-path cleanup still follows. Whether this alias is reachable in a valid deployment is unresolved.

If the addresses differ, the observable sequence is:

1. F-e1c941072f1b receives the destination-slot address.
2. After that call returns, the local output word is read and copied into the destination.
3. The local output word is read for a nonzero guard.
4. If that guard observes zero, the output-related referent sequence is skipped.
5. Otherwise, the then-current local output word is used to access a 32-bit field at referent offset `+8`.
6. If that field observation is nonzero, F-338d16f9ce72 is called without explicit arguments.
7. After that optional call, the local output word is read again for the referent access that increments the 32-bit field at `+4`.
8. The subsequent `+8` check also uses the local output word as then read. If the field observation is nonzero, F-1333331b2da1 is called without explicit arguments.

The value produced immediately by construction is not necessarily the value copied into the destination. Destination cleanup occurs first, and its preservation of the local output slot is unknown.

Likewise, the initial nonzero guard does not pin a referent throughout the branch. Construction received the output-slot address, and dependency effects on that storage are unknown. In particular, F-338d16f9ce72 is not established to preserve the slot.

There is **no repeated nonzero guard after F-338d16f9ce72**. Normal continuation requires the later slot-derived accesses to remain valid. If the slot changes to zero or another unusable value, the earlier guard does not protect those dereferences.

Consequently, the destination copy, first `+8` check, `+4` increment, and second `+8` check are not established to concern one fixed referent. The two conditional helpers are not established to form a matched synchronization pair.

### Successful cleanup and common exit

After destination handling, the function:

1. Calls F-e1c941072f1b with the local output-slot address.
2. Calls F-fe0481c27e53 with the local input-slot address.
3. Selects one as the normal result.
4. Overwrites the descriptor’s first word with the saved **first** global observation.
5. Calls F-22abefe7f288 with the descriptor address.
6. Calls F-d49b39475920 with the object.
7. Returns one if all reached calls complete normally.

Local output cleanup receives the slot in its then-current state, not necessarily the word originally produced by construction or copied to the destination.

Descriptor restoration does not reread the global word and does not use the second observation that initialized the descriptor. Only the first descriptor word is explicitly restored; the second retains its then-current contents.

Every visible normal-return path calls F-d49b39475920. The successful path does not make the second F-b7522c184a8b call used by unsuccessful main-body paths.

## Side effects

Direct writes include:

- The option byte and interrogation-result field.
- Object handle storage through dependencies receiving its address.
- The destination word on the unequal-address branch.
- Local descriptor and handle storage.
- Reached referent fields at `+4`.

Dependencies receive addresses into object, caller, and local storage. Their effects and preservation guarantees are unavailable. A dependency without explicit arguments may still affect previously exposed storage or global state.

The increments resemble retention operations, and several dependencies resemble cleanup or assignment. These are inferred roles, not ownership guarantees. The function alone does not establish allocation, release, transfer, borrowing, count decrement, or destination lifetime.

The incremented fields are 32 bits. There is no visible overflow check, saturation path, or local overflow error. Defined overflow behavior is unresolved. Signed comparison applies to preparation selection; the increment operations do not establish valid count ranges.

No atomicity or concurrency-safety guarantee is visible. The representation does not establish a finer evaluation order for constituent reads within an increment expression.

There is no visible rollback around failed preparation, construction, destination replacement, or exceptional dependency exits. A reset-like call is not proof that previous writes are undone. Nonlocal exits may bypass later cleanup and the common exit.

## Dependencies

All roles below are limited to visible use.

| Opaque ID | Established use |
|---|---|
| F-b7cd573c0b4c | Called without explicit arguments before the gate read. |
| F-d49b39475920 | Receives the object on every visible normal-return path. |
| F-b7522c184a8b | Receives the object before setup and again on unsuccessful main-body exits. |
| F-e4b7a6dcc8e5 | Receives object handle-storage and caller input-slot addresses. |
| F-2e1e5059bccf | Receives temporary storage and zero; returns a pointer whose first word is read. |
| F-fe0481c27e53 | Receives the comparison temporary and, later, the local input slot. |
| F-56a66efac853 | Supplies receivers for two separately resolved indirect calls. |
| F-dcc406073ef9 | Receives the object and selected preparation value. |
| F-34ab68c7ba90 | Supplies the construction context. |
| F-a817d8d58c10 | Called for zero context; marked nonreturning. |
| F-962cf82b6591 | Receives output-slot address, context, descriptor address, and input-slot address. |
| F-e1c941072f1b | Receives the destination conditionally and local output slot afterward. |
| F-338d16f9ce72 | Conditionally called between the first output-related `+8` check and increment. |
| F-1333331b2da1 | Conditionally called after the later `+8` check. |
| F-22abefe7f288 | Receives the descriptor after first-word restoration. |

The indirect dependencies are selected at dispatch-table byte offsets `+8` and `+0x30`. Their opaque identities and implementations are unavailable.

## Acceptance examples

These are conditional specifications, not executed tests. Normal-return examples require valid reached accesses and normally returning dependencies, except for the explicitly nonreturning case.

1. **Gate set after entry.** A nonzero gate observation skips the main body, calls F-d49b39475920, and returns zero. It does not exclude dependency side effects.

2. **Captured words equal.** Setup and comparison-temporary cleanup occur before the equality decision. A second F-b7522c184a8b call follows, then the common zero-result exit. Neither indirect method is reached.

3. **Interrogation returns zero.** Object field `0x2B8` receives zero. Preparation is skipped; the unsuccessful-path reset-like call and common exit follow.

4. **Signed selection.** Method result 12 with stable bound 5 selects 5. Stable bound 20 selects 12. Method result −3 with stable bound 4 selects −3.

5. **Different bound observations.** If the retained bound is 9, the comparison-time bound is 2, and the method result is 5, preparation receives 9.

6. **Option width.** Option `0x000001AB` stores byte `0xAB`; the first indirect method receives all 32 bits.

7. **Preparation returns zero.** Context acquisition is skipped. Earlier writes are not locally rolled back.

8. **Zero context.** The first global read still occurs, followed by F-a817d8d58c10. No second global read, descriptor initialization, or normal return is established.

9. **Different global observations.** If the first observation is `0x11111111` and the second is `0x22222222`, descriptor initialization uses `0x22222222`. Final first-word restoration uses `0x11111111`.

10. **Late input modification.** If an earlier dependency changes the caller’s slot, late capture uses its current word. A nonzero capture triggers the input-related `+4` increment.

11. **Zero output through destination handling.** On the unequal-address branch, if the local output word is zero after destination cleanup and remains zero through copying and the guard, zero is copied and the output-related increment is skipped. Successful cleanup can still return one. Zero immediately after construction alone is insufficient.

12. **Destination cleanup changes output.** If construction establishes zero but F-e1c941072f1b changes the local output slot to a valid nonzero handle before the copy, that later word is copied. If the guard also observes it as nonzero, referent processing is entered.

13. **Helper retargets the slot.** Suppose the guard and first `+8` check use valid handle A, whose checked field is nonzero. If F-338d16f9ce72 changes the output slot to valid handle B, later slot-derived accesses may use B. The increment is not guaranteed to affect A or the handle already copied into the destination.

14. **Helper leaves an unusable slot.** If F-338d16f9ce72 changes the slot to zero, no second nonzero guard skips the increment. Normal completion is not guaranteed; subsequent dereferences still require valid targets.

15. **Independent helper decisions.** With valid accesses, nonzero first `+8` and zero second `+8` observations call only F-338d16f9ce72. The reverse observations call only F-1333331b2da1. These observations need not concern the same referent.

## Uncertainties

The following remain **documented_only**:

- Resource types, option meanings, quantity units, and valid ranges.
- Complete calling conventions, alignment, and dispatch-table definitions.
- Global-word meaning, mutability policy, and concurrency model.
- Whether differing repeated observations occur in valid deployments.
- Descriptor semantics and the significance of restoring only its first word.
- Every dependency’s ownership, release, retention, and preservation contract.
- Whether dependencies retain exposed local-slot addresses.
- Whether referent fields are reference counts or conditional helpers provide synchronization.
- Defined overflow behavior.
- Exceptional exits and the zero-context failure mechanism.
- Whether reset-like calls restore particular invariants.
- Reachability of unusual aliasing and output-slot retargeting.

Unknown preservation must not be replaced with a fixed-referent assumption, especially across destination cleanup or F-338d16f9ce72.

## Evidence

Established behavior derives from the supplied static representation: argument uses, access widths and offsets, ordered calls, captured values, repeated reads, signed comparison, branch conditions, direct stores, and visible return selection.

Inferred intent is limited to role descriptions such as handle, preparation, construction, retention-like increment, and cleanup.

The output is read after destination cleanup and reread for later referent accesses. No post-helper nonzero guard is visible. The saved first global observation, second initialization observation, and final restoration are also distinct operations.

Dependency implementations and external ownership contracts were unavailable. No runtime tests have been performed.
