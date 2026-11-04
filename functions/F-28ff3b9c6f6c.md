<!-- Trusted provenance: F-28ff3b9c6f6c; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-28ff3b9c6f6c

## Purpose

F-28ff3b9c6f6c conditionally processes a referenced object, selects one of two dependency paths from its flags, and returns a 32-bit object field only if the object's final state byte matches a mode sampled at entry.

The established behavior is a flag-dependent dispatch followed by a state check. Interpreting the dependencies as initialization, cleanup, activation, or resource management is not supported by the supplied evidence.

Three distinctions are central to the contract:

- The mode is selected before the reference is checked for zero.
- Some decisions use the initially observed flags; a later dispatch decision uses freshly read flags.
- A nonzero reference and successful dependency calls do not by themselves produce a nonzero return. The final state must match, and the returned field may itself contain zero.

## Interface

### Input

The function accepts one argument represented in the supplied static evidence as a 32-bit integer used as an object address.

- Zero is treated as an absent object.
- A nonzero value is used for fixed-offset memory accesses.
- No object length, type tag, accessibility check, or alignment check is visible.
- The function does not establish that every nonzero argument is valid.

The description below uses “object reference” for this argument without asserting a language-level pointer type or ownership model.

### Directly accessed object layout

Offsets are measured in bytes from the supplied object reference.

| Offset | Width | Access | Meaning established by use |
|---|---:|---|---|
| `+0x14` | 32 bits | Conditional read | Value returned after a matching final state |
| `+0x1c` | 16 bits | Reads and one conditional write | Flags controlling eligibility, flag propagation, and dispatch |
| `+0x1f` | 8 bits | Conditional read | Final state compared against the selected mode |

The flags occupy bytes `+0x1c` and `+0x1d`; the state byte at `+0x1f` does not overlap them. This function has no direct access to byte `+0x1e`.

The evidence establishes access widths and relative offsets, not byte order, packing rules for a larger structure, or a complete object definition.

### External input

An externally addressed one-byte mode source is read at entry:

- A byte equal to zero selects mode **3**.
- Any nonzero byte selects mode **2**.

The address resolution and lifetime of this external storage are not specified. Its byte value is sampled once for this invocation. Later dependency changes to that storage do not cause the function to select a different mode.

### Output

The function returns a 32-bit value:

- Zero for an absent object.
- Zero when the initial flags contain mask `0x0001`.
- Zero when the final state byte differs from the selected mode.
- Otherwise, the 32-bit value read from object offset `+0x14`.

No separate status or error object is returned. A zero result is ambiguous: it can be a rejection result or a successfully selected object field whose contents are zero. The signedness or higher-level interpretation of the returned bits is not established.

### Preconditions

For a normal completion, the entry mode source must be readable even when the object reference is zero.

For a nonzero object reference, memory must support whichever accesses the selected path reaches, together with the requirements of any called dependencies. The visible function performs no bounds, lifetime, or alignment validation. Invalid accesses are not shown being converted into a zero result.

## Behavior

### Entry sampling and eligibility

The external mode byte is read first. The function derives mode 3 or mode 2 from that observed value and retains the selected mode for subsequent calls and the final comparison.

Only after this sampling does the function check whether the object reference is zero. A zero reference returns zero without direct object access and without invoking any of the listed dependencies.

For a nonzero reference, the function reads the 16-bit flags at `+0x1c`. If mask `0x0001` is set in that initial value, it returns zero. On this path there is:

- No direct flags write.
- No dependency invocation.
- No direct state-byte read.
- No direct return-field read.

The entry mode-source read still occurs.

### Flags update

When the initial flags have mask `0x0001` clear, the function writes a 16-bit value back to `+0x1c`.

The written value preserves every bit of the initial flags and additionally sets mask `0x0002` if mask `0x0010` was initially set.

Consequently:

- Initial mask `0x0010` implies mask `0x0002` is set by this write.
- An already set mask `0x0002` remains set.
- Mask `0x0010` itself is not cleared.
- No bit is cleared by this direct write.
- The write occurs even when the resulting 16-bit value equals the initially read value.

This is a 16-bit flag operation, not an increment or a wider arithmetic update. The extracted contribution is either zero or `0x0002`; it cannot overflow the 16-bit destination.

The value written is derived from the earlier flags observation. No fresh flags read precedes this write.

### Optional preliminary dependency

After the flags write, the function tests mask `0x0800` in the **initially read** flags.

If that mask was set, it calls F-be57f04e6ec4 with the original object reference as its sole visible argument. Otherwise, this dependency is skipped.

This decision does not use a post-write or post-call flags observation. The dependency's return value is not used.

The call occurs before the dispatch described below. Its effects are unknown: it may alter the object, external state, or other accessible memory. Its name and position do not establish that it frees, retains, validates, or preserves anything.

### Dispatch using current flags

After the optional preliminary call, the function reads the 16-bit flags at `+0x1c` again. The dispatch decision uses mask `0x0002` from this new observation.

If that mask is clear:

- F-e6e9e68a0990 is called with the original object reference and the selected mode.
- F-98085658bfd4 and F-94f17d1d24a0 are not called by this dispatch.

If that mask is set:

- F-98085658bfd4 is called with the original object reference and the selected mode.
- After that call returns, F-94f17d1d24a0 is called with the original object reference.
- There is no intervening direct state or flags check that conditions the second call.

All dependency return values are ignored by the visible caller. This establishes only that their return values do not select the caller's path; it does not establish whether those dependencies return status values or can fail by other means.

The fresh flags read is significant. In particular, F-be57f04e6ec4 can affect the dispatch if it changes mask `0x0002`. Conversely, the initial eligibility test for mask `0x0001` is not repeated: setting that bit later does not, by itself, cause this caller to skip dispatch or the final state check.

### Final state and return

After the selected dependency sequence returns, the function reads the byte at object offset `+0x1f`.

- If that byte equals the retained mode, the function reads and returns the 32-bit field at `+0x14`.
- If it differs, the function returns zero without directly reading the field at `+0x14`.

The expected byte is exactly 2 or 3. Signed versus unsigned interpretation of the byte does not change equality with either of these values.

On the two-call dispatch path, this comparison occurs after F-94f17d1d24a0, not between the two calls. Thus, the result depends on state visible after the complete selected sequence.

The returned field is read at the end. It is not snapshotted before dependency calls. Dependency changes to that field can therefore affect the returned value.

## Side effects

### Direct observable access order

For a valid eligible object and normally returning dependencies, the caller's direct accesses and calls are ordered as follows:

1. Read the external mode byte.
2. Read the initial 16-bit flags.
3. Write the updated 16-bit flags.
4. Optionally call F-be57f04e6ec4, based on initial mask `0x0800`.
5. Read the current 16-bit flags.
6. Execute the selected dependency sequence.
7. Read the final state byte.
8. Only if the state matches, read the 32-bit return field.

The zero-reference and initial-ineligibility paths terminate earlier as described above.

This ordering describes the supplied static representation. It does not establish volatile semantics, atomicity, memory barriers, or a concurrency protocol.

### Mutation and rollback

The caller's only visible direct object mutation is the 16-bit flags write. It does not directly modify the state byte or the returned field.

A later state mismatch does not undo the flags write or any dependency effects. Zero return must not be interpreted as “no state change.”

The function contains no visible allocation, deallocation, ownership transfer, or reference-count operation of its own. Dependencies remain unconstrained in those respects.

### Lifetime and aliasing

The same original object reference is passed to dependencies and used again afterward. No replacement reference is obtained from their return values.

Normal completion therefore requires the storage needed by subsequent accesses to remain valid. The visible caller does not enforce that requirement. Whether a dependency may invalidate the object, and whether surrounding invariants prevent that, is unknown.

No non-aliasing guarantee is established between the external mode source and object storage. The entry mode is nevertheless fixed from the first read; later writes to aliased storage would not resample it.

## Dependencies

| Opaque ID | Invocation condition | Visible arguments | Use of result |
|---|---|---|---|
| F-be57f04e6ec4 | Eligible object; initial flags contain `0x0800` | Original object reference | Ignored |
| F-e6e9e68a0990 | Eligible object; reread flags lack `0x0002` | Original object reference, selected mode | Ignored |
| F-98085658bfd4 | Eligible object; reread flags contain `0x0002` | Original object reference, selected mode | Ignored |
| F-94f17d1d24a0 | After F-98085658bfd4 returns | Original object reference | Ignored |

The mode arguments carry numeric value 2 or 3. Their exact ABI argument width is not established by the supplied representation.

No dependency contract is available for global preservation, object ownership, memory release, error signaling, termination, or mutations. All descriptions of accesses after a dependency call are conditional on that call returning normally and the accessed storage remaining valid.

## Acceptance examples

These are conditional static contract examples, not executed tests. Unless stated otherwise, they assume valid storage, no concurrent interference, and normally returning dependencies.

1. **Absent object still samples mode.**  
   Given a zero object reference and a readable external mode source, the function reads that source and returns zero. It performs no direct object accesses and calls no listed dependency.

2. **Initial mask `0x0001` rejects processing.**  
   Given initial flags `0x0811`, the function returns zero after reading the flags. Neither `0x0010` nor `0x0800` causes an update or call because initial eligibility fails.

3. **Clear dispatch mask selects the single-call path.**  
   Given mode-source byte zero and initial flags `0x0000`, the selected mode is 3 and the direct flags write stores `0x0000`. If the subsequent flags read remains zero, only F-e6e9e68a0990 is called, with mode 3. If the resulting state byte is 3, the final 32-bit field is returned.

4. **Mask `0x0010` promotes the dispatch mask.**  
   Given a nonzero mode-source byte and initial flags `0x0010`, the selected mode is 2 and the direct flags write stores `0x0012`. With those flags unchanged until dispatch, F-98085658bfd4 is followed by F-94f17d1d24a0.

5. **The preliminary call can redirect dispatch.**  
   Given initial flags `0x0810`, the caller first writes `0x0812` and calls F-be57f04e6ec4. If that dependency clears `0x0002`, and it remains clear through the following read, dispatch selects F-e6e9e68a0990 despite the earlier promotion.

6. **Late ineligibility does not repeat the entry gate.**  
   Given an initially eligible object requiring the preliminary call, suppose that call sets `0x0001`. The caller still proceeds to its fresh dispatch-mask read and selected dependency sequence. It does not repeat the initial rejection test.

7. **The last dependency determines the observed final state.**  
   On the two-call path with mode 2, suppose F-98085658bfd4 leaves state 2, but F-94f17d1d24a0 changes it to 3. The function returns zero and does not directly read `+0x14`.

8. **Matching state does not guarantee a nonzero result.**  
   Given a final state matching the selected mode and a 32-bit field of zero, the function reads that field and returns zero. This is observationally identical at the return-value level to rejection, though its accesses and side effects differ.

9. **External mode changes do not change the retained expectation.**  
   Given an entry mode-source byte of zero, mode 3 remains the argument and final comparison value even if a dependency changes the external byte to nonzero.

## Uncertainties

The following are **documented_only**:

- The domain meaning of the object, flags, mode source, state byte, and returned field.
- The complete object size, byte order, alignment rules, and target ABI.
- Whether the return field denotes a pointer, handle, number, or other opaque bits.
- Dependency effects, failure conventions, ownership behavior, and preservation guarantees.
- Whether dependencies can invalidate the object or terminate without returning.
- Any synchronization mechanism or supported behavior under concurrent mutation.
- The consequences of invalid references, inaccessible memory, or unsupported alignment.
- Behavior outside the supplied reachable representation.

**Inferred intent:** propagating `0x0010` into `0x0002` appears to influence which processing path is taken, while the final state comparison appears to qualify the returned field. These are interpretations, not additional requirements.

## Evidence

This contract is based solely on the supplied static function representation and the provided opaque dependency mapping.

Established evidence includes the access widths and offsets, entry mode selection, initial flag gate, flags update, distinction between initial and reread flags, dependency call conditions and order, final state comparison, and conditional 32-bit return-field read.

The supplied material includes notices about removed unreachable blocks but does not provide those blocks. Their contents and significance are not established and add no behavior to this contract.

No dependency implementations, runtime traces, external specifications, or tests were supplied. No runtime tests have been performed.
