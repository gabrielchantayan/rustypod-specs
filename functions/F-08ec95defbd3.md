<!-- Trusted provenance: F-08ec95defbd3; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-08ec95defbd3

## Purpose

F-08ec95defbd3 coordinates a 40-byte allocation request, initialization of the returned object, and submission of a temporary descriptor. A nonzero optional input causes an additional dependency call whose result is written into that descriptor.

The function distinguishes three outcomes:

- **Submission reports success:** clear bit 10 in a 32-bit field at byte offset 8 of the object identified through the output slot.
- **Submission reports failure:** invoke two dependencies, in order, with the recorded intermediate-object address.
- **An earlier status is nonzero:** invoke a release-role dependency with the original source value instead.

These are established control-flow distinctions, not proof of resource ownership. The dependency contracts do not establish what the objects represent, whether either cleanup call releases memory, or whether successful submission transfers ownership.

All behavior below is derived from static evidence. No runtime tests have been performed.

## Interface

### Inputs and return value

The function has six input positions and a 32-bit status result.

| Position | Contract name | Established use |
|---|---|---|
| 1 | Context pointer | Dereferenced to capture its first 32-bit word before validation. Also passed directly to the optional-value and submission dependencies. |
| 2 | Source value | A 32-bit value checked for zero, forwarded to initialization, and supplied to the release-role dependency on earlier error. |
| 3 | Initialization argument | An opaque 32-bit value forwarded to initialization. |
| 4 | Submission argument | An opaque 32-bit value forwarded to submission. |
| 5 | Optional value | A 32-bit value tested for zero. When nonzero, it is passed to the optional-value dependency. |
| 6 | Output slot | Pointer to a 32-bit slot passed to submission. After submission reports success, its contents are used as an object address for a field update. |

The source and optional values have integer-like representations, but their semantic types are unknown. They may represent handles, addresses, or other values; this contract does not choose among those interpretations.

Addresses, stored values, and field widths follow the supplied 32-bit representation. A 64-bit interpretation is not established.

The selected return status is one of:

- 33 from the explicit null-context branch, if execution reaches it;
- 6 from the zero-source branch;
- the value read from the allocation-status storage, including a later reread after initialization;
- the result returned by submission.

Zero selects the next success-oriented stage. No meaning is established for other dependency-provided status values. Cleanup return values do not replace the selected status.

### Memory and dependency preconditions

Ordinary execution requires the initial context-word read to complete. The later null check does not protect that access.

Additional requirements depend on the selected path:

- The allocation-status storage must contain a defined 32-bit value when read. No local initialization of that storage precedes the allocation call.
- After allocation reports zero, the returned address must remain suitable for initialization and the subsequent 32-bit write at byte offset 24.
- After submission reports zero, the output slot and the target field at byte offset 8 must support the required reads and write.
- Every invoked dependency must meet its own undisclosed validity and lifetime requirements.

The function does not independently validate the returned allocation address, output slot, output object, or optional dependency result.

## Behavior

### Initial captures precede validation

The function first reads a 32-bit value from a fixed global location. It then reads the first 32-bit word through the context pointer.

The captured global value is later used for an intermediate-object field write. The captured context word is used as the first argument to allocation and to the release-role dependency.

Both reads precede the explicit input checks. Therefore:

- A null context does not prevent the attempted context dereference.
- A zero source does not prevent either initial read.
- No later explicit reread of the global location or context word replaces these captured values.

The roles “initialization metadata” and “allocation domain” are plausible interpretations of these values, not established semantic types. Dependencies receiving the context pointer may independently inspect its current contents.

### Input checks and allocation

After the initial reads, the function checks the context pointer and then the source value.

A null context selects status 33. Otherwise, a zero source selects status 6. Either nonzero status leads to the earlier-error path described below.

For a nonnull context and nonzero source, F-3ce007b692cc receives:

1. The captured context word.
2. A size request of 40 bytes.
3. A pointer to temporary status storage.

Its 32-bit return value is treated as the intermediate-object address. Its status output, rather than that returned address, determines whether initialization proceeds.

Immediately after the call, the function sets its recorded intermediate-object cleanup value to zero and reads the temporary status. A nonzero status skips initialization and submission. A zero status permits initialization without a separate null-address or range check.

The allocation dependency receives a pointer into a temporary region that subsequently contains the submission descriptor. The evidence does not establish how much of that region the dependency writes or whether it retains the pointer.

### Initialization and status reread

When the first allocation-status observation is zero:

1. F-8f073f7099e1 receives the returned address, source value, and initialization argument.
2. The function writes the captured global value as a 32-bit word at byte offset 24 of the returned object.
3. It records the returned address for descriptor construction and possible cleanup.
4. It reads the allocation-status storage again.

The initializer’s return value is unused. There is no separate initializer-success check.

The second status observation controls whether descriptor construction occurs. Although the initializer is not directly passed the status pointer, the supplied evidence does not exclude access through retained pointers, aliasing, or other dependency effects.

If the second observation is nonzero, the function takes the earlier-error path. That path supplies the original source value to the release-role dependency; it does not invoke the two-call intermediate-object cleanup sequence.

### Descriptor construction and optional derivation

When the selected status is zero, the function writes selected fields of a temporary descriptor.

Offsets below are relative to the descriptor pointer passed to F-589adbc0f0a7. They describe **explicit writes by this function**, not unconditional guarantees about what the submission dependency will observe.

| Byte offset | Width | Explicit local action |
|---|---:|---|
| 0 | 32 bits | Write selector 2. If the optional input is nonzero, overwrite it with 10 before calling F-17be7b6b79ad. |
| 4 | 32 bits | No explicit descriptor-field write is shown. |
| 8 | 32 bits | No explicit descriptor-field write is shown. |
| 12 | 32 bits | No explicit descriptor-field write is shown. |
| 16 | 32 bits | Write the recorded intermediate-object address before the optional dependency call. |
| 20 | 32 bits | For a nonzero optional input, write the return value of F-17be7b6b79ad after that call returns. Otherwise, no explicit write is shown. |

The descriptor starts four bytes after the allocation-status location within the same temporary region. Its full type, required extent, and interpretation by submission are unknown.

For a zero optional value, the function does not call F-17be7b6b79ad.

For a nonzero optional value, the observable sequence is:

- Write selector 10.
- Call F-17be7b6b79ad with the context pointer and optional value.
- Write its 32-bit result at descriptor offset 20.
- Call submission.

There is no success or failure test on the optional result. Zero is submitted as data just as a nonzero result is.

**The optional call can invalidate assumptions about previously written fields.** F-3ce007b692cc previously received a pointer into this same temporary region and could have retained it. The evidence does not exclude F-17be7b6b79ad obtaining access through that retained pointer and changing the selector, the descriptor’s object-address field, or other contents.

In particular, selector 10 is not rewritten after the optional dependency returns. Submission therefore observes selector 10 only if intervening effects leave that field unchanged. The same qualification applies to the previously written intermediate-object address. This limitation concerns initialized fields as well as fields lacking explicit initialization.

The write at offset 20 occurs after the optional call, so it replaces any value that call left in that specific word before returning. It does not restore other fields.

### Submission

F-589adbc0f0a7 receives:

1. The original context pointer.
2. The temporary descriptor pointer.
3. The submission argument.
4. The output-slot pointer.

Its 32-bit return value becomes the selected status.

The descriptor resides in invocation-local storage. The evidence establishes no lifetime guarantee after this invocation returns. Any retained use would require an external lifetime arrangement not shown here.

### Submission success

If submission returns zero, the function accesses the output slot and uses its contents to address a 32-bit field at byte offset 8.

The field transformation clears bit 10, whose value is 1,024, while preserving the other 31 bits. The 32-bit mask is hexadecimal FFFFFBFF.

The represented expression references the output slot for both the field read and the field write. The evidence does not establish a single captured target address or a finer machine-level ordering of those references. For stable, ordinary storage, the effect is a read-modify-write of the identified field.

No local validity check precedes these accesses. Reported submission success with an invalid output slot or target can therefore fail during post-submission memory access rather than return an ordinary error status.

Neither intermediate-object cleanup dependency is called on this path. That omission does not prove ownership transfer.

### Submission failure

If submission returns nonzero, the function calls:

1. F-22f99af8431f with the recorded intermediate-object address.
2. F-bf7242c98d06 with the captured context word and that recorded address.

The second call is not conditional on a result from the first. No cleanup result replaces the submission status.

These arguments use the recorded address, not a reread of descriptor offset 16. Thus, changing the descriptor’s object-address field does not itself change which recorded address this cleanup path supplies.

The function performs no explicit output-slot restoration or clearing and no local output-field bit update on failure.

### Earlier error

For a nonzero status selected before descriptor construction, F-bf7242c98d06 receives the captured context word and the original source value. The selected status is then returned.

This includes:

- The explicit input-error branches, insofar as execution reaches them after the initial reads.
- A nonzero first allocation-status observation.
- A nonzero second status observation after initialization.

In the last case, initialization and the offset-24 write have already occurred, but the visible cleanup still targets the source value rather than the intermediate object.

## Side effects

The function directly performs or requests the following effects:

- Reads the fixed global value and the context’s first word before validation.
- Exposes temporary storage to allocation.
- Writes the captured global value at intermediate-object offset 24.
- Writes selected descriptor fields.
- Exposes the descriptor and output slot to submission.
- On reported submission success, reads and writes the output object’s 32-bit field at offset 8.
- Invokes the path-specific dependency sequences described above.

The principal ordering is allocation, initialization, object-field write, descriptor writes, optional derivation if selected, submission, then either the output-field update or failure cleanup.

Dependency effects remain unconstrained except where their arguments and results are visible. They may change globals, context storage, retained temporary storage, or resources. No assumption that they preserve memory or release ownership is justified.

In particular, neither cleanup call’s lifetime effect is known. The first submission-failure cleanup call might alter the resource before the second receives the same recorded address. The function nevertheless makes both calls if they return normally.

No atomicity, synchronization, rollback, or exception-handling behavior is established. The bit-clear operation is not shown to be atomic.

## Dependencies

| Opaque function ID | Established interaction |
|---|---|
| F-3ce007b692cc | Receives the captured context word, size 40, and a temporary status pointer. Returns the value used as an object address. May have access to temporary storage later reused for the descriptor; retention is not excluded. |
| F-8f073f7099e1 | Receives the returned object address, source value, and initialization argument. Its return value is ignored. |
| F-17be7b6b79ad | Called only for a nonzero optional input, after selector 10 and the descriptor’s object-address field have been written. Its result is subsequently written at descriptor offset 20. |
| F-589adbc0f0a7 | Receives context, descriptor, submission argument, and output slot. Its status selects the output-field update or intermediate-object cleanup. |
| F-22f99af8431f | First dependency called after nonzero submission status, with the recorded intermediate-object address. |
| F-bf7242c98d06 | Receives the captured context word and either the source value on earlier error or the recorded intermediate-object address after submission failure. |

“Allocation,” “initialization,” “submission,” and “cleanup” are role descriptions inferred from usage. They are not independently established dependency contracts.

## Acceptance examples

These are conditional static acceptance criteria, not executed tests. All assume the stated memory accesses and dependency calls complete normally.

1. **Zero source with readable context.**  
   The initial global and context-word reads occur. Status 6 is selected. F-bf7242c98d06 receives the captured context word and zero. Allocation and submission are skipped; the returned status is 6.

2. **Null context.**  
   The context dereference precedes validation. Returning 33 is therefore not unconditional. If that access completes and execution reaches the explicit branch, status 33 is selected and the earlier-error dependency is called before return.

3. **Allocation reports status 7.**  
   For a nonzero source, if the first status observation is 7, initialization and submission are skipped. F-bf7242c98d06 receives the source value, and the function returns 7 regardless of the allocator’s returned address.

4. **Successful submission without optional data.**  
   With zero status through initialization and a zero optional input, the function writes selector 2 and the recorded address at descriptor offset 16. It does not call F-17be7b6b79ad. If submission returns zero and a stable valid output target contains hexadecimal FFFFFFFF at offset 8, that field becomes hexadecimal FFFFFBFF.

5. **Optional data without intervening descriptor mutation.**  
   For a nonzero optional input, the function writes selector 10 before calling F-17be7b6b79ad. If intervening effects leave the selector and object-address field unchanged, submission receives those values. The optional result is written at offset 20 even when it is zero; that zero does not itself prevent submission.

6. **Optional dependency changes an initialized field.**  
   Suppose allocation retains access to the temporary region and the optional dependency uses that access to replace selector 10 with 14. If no later effect changes it, submission receives selector 14: this function does not restore 10 after the optional call. A change to offset 16 is likewise not locally restored.

7. **Submission reports status 9.**  
   F-22f99af8431f receives the recorded intermediate-object address, followed by F-bf7242c98d06 with the captured context word and that address. No local bit-clear occurs. The function returns 9.

8. **Bit already clear.**  
   With successful submission and stable valid output storage, hexadecimal 00000020 remains numerically unchanged. The represented success path still reads and writes the field.

9. **Status changes after initialization.**  
   If the first allocation-status observation is zero and the second is 12, initialization and the offset-24 write occur, but submission is skipped. The earlier-error call receives the source value. The function returns 12 without the two-call intermediate-object cleanup sequence.

## Uncertainties

The following remain **documented_only**:

- Exact source-level types, given the supplied type-propagation warning.
- Resource identities, ownership transfers, and actual cleanup semantics.
- Whether allocation always initializes its status output.
- Whether any dependency retains or shares access to temporary storage.
- Actual descriptor contents after intervening dependency effects, including explicitly initialized fields.
- Which descriptor fields submission reads and what extent it requires.
- Optional-result meaning, ownership, and error representation.
- Output validity following a zero submission status.
- Aliasing restrictions among inputs, temporary storage, globals, and output storage.
- Behavior under concurrent mutation, faults, exceptions, or reentrancy.
- Address-overflow behavior for field offsets in invalid or boundary addresses.
- Whether the pre-check null dereference faults or permits execution to reach the explicit status-33 branch.

## Evidence

**Established static evidence:** The supplied representation establishes the initial read order, six input positions, 32-bit data operations, statuses 33 and 6, 40-byte request, repeated allocation-status observation, field offsets, descriptor-write order, dependency argument flow, cleanup order, and mask FFFFFBFF.

It also establishes that selector 10 and the descriptor’s object-address field are written before the optional dependency call, while the optional result is written afterward. No intervening preservation guarantee or subsequent restoration of those earlier fields is shown.

**Inferred intent:** Object construction, descriptor submission, allocation-domain use of the context word, and cleanup-oriented dependency roles follow from usage patterns only.

**Verification boundary:** No runtime tests have been performed. Acceptance examples describe conditional behavior supported by static evidence, not observed execution.
