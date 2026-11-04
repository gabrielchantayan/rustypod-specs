<!-- Trusted provenance: F-e0a692eba8b7; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-e0a692eba8b7

## Purpose

F-e0a692eba8b7 conditionally decrements a 32-bit field in an object referenced by a caller-supplied slot. A resulting value of zero selects a cleanup path; any other resulting value selects a shorter completion path. Every explicit normal-return path ends with the caller’s slot containing zero.

The layout and decrement are consistent with releasing a reference-counted object. That interpretation is **inferred intent**, not an established ownership contract. The supplied evidence does not establish allocation provenance, dependency semantics, synchronization guarantees, or whether all cleanup calls actually return.

Several dependency calls are explicitly annotated as non-returning. Under those annotations, some cleanup paths terminate or transfer control before the caller’s slot is cleared. Those paths must not be described as successful normal returns.

## Interface

### Input

The function receives one pointer to a mutable slot. The slot contains a 32-bit object address or zero.

A valid slot is required even when its contents are zero: the function reads the slot before deciding whether to return. There is no explicit check for a null or otherwise invalid slot address.

The supplied representation uses 32-bit integer and address-sized values. This contract describes that representation; it does not generalize the layout to a 64-bit address space.

### Referenced layouts

The object address exposes these fields:

| Byte offset | Width | Observed use |
|---|---:|---|
| 0 | 32 bits | Address of an optional two-word record |
| 4 | 32 bits | Value decremented and compared with zero |
| 8 | 32 bits | Optional address examined during a later cleanup stage |

The optional record at offset 0 exposes:

| Byte offset | Width | Observed use |
|---|---:|---|
| 0 | 32 bits | First argument to F-1d060f30d5b5 |
| 4 | 32 bits | Second argument to F-1d060f30d5b5 |

These accesses establish only the fields used here, not complete object definitions, alignment requirements, or the absence of other fields.

The record’s two words are forwarded as values. Their interpretation as addresses, sizes, handles, or another pair of values is unknown.

### Output

No return value is specified. Observable results consist of memory writes, dependency invocations, and whether control returns.

On an explicit normal return:

- An initially zero slot remains zero, with no dependency calls.
- A path that begins with a nonzero slot reaches a final write of zero to that slot.

The function has no explicit status code, exception translation, or reported error result.

### Preconditions and ownership limits

For each path reached, all directly accessed storage must be valid for the corresponding read or write. Dependency calls may impose additional requirements that are not supplied.

The function does not establish that its input is a unique reference, that the offset-4 value is positive, or that the offset-8 field is initialized on paths that reach it.

Calls to F-feee9156a257 resemble release or disposal operations, but their ownership effects are unknown. In particular, this contract does not assume that they free memory, preserve globals, leave the caller’s slot intact, or return.

## Behavior

### Initial gate

The first operation reads the caller’s slot.

If the value is zero, the function returns immediately. It does not invoke any dependency, decrement a field, or perform the final slot write on this path.

If the value is nonzero, the function invokes F-c69e18c9c6b7 with no explicit arguments shown.

The object address used afterward is obtained from the slot after that call. It is not established that the initial nonzero address is retained. If F-c69e18c9c6b7 changes the slot, later operations follow the changed contents. No new zero check precedes the decrement.

### Decrement and branch

After F-c69e18c9c6b7 returns, the function:

1. Reads the 32-bit value at offset 4 of the object then referenced by the slot.
2. Computes that value minus one.
3. Writes the computed value to offset 4 through the slot.
4. Tests the computed value for equality with zero.

The object address is accessed through the slot for both the read and the write. There is no dependency call between those operations, but the supplied evidence does not establish atomicity or protection against concurrent mutation.

The test uses the computed value, not a fresh read of the stored field. A later change to that field therefore does not change the branch decision already being made.

The field and intermediate result are represented as 32-bit signed integers in the supplied material. Ordinary representable cases are clear: one becomes zero, zero becomes negative one, and two becomes one. The treatment of signed overflow at the minimum signed 32-bit value is not established by this source representation. A portable overflow rule must not be invented.

There is no range check or underflow guard. In particular, a field value of zero does not itself select cleanup: decrementing zero produces negative one in the ordinary representable case, which selects the nonzero-result path.

### Nonzero-result path

If the computed result is not zero:

- F-afd06380d3b7 is invoked with no explicit arguments shown.
- If that call returns normally, the caller’s slot is overwritten with zero.
- The function then returns.

No offset-0 record or offset-8 field is explicitly inspected on this path.

The final write targets the original caller-supplied slot. Consequently, if F-afd06380d3b7 changes that slot and returns, its replacement value is also overwritten with zero.

### Zero-result path: optional record

If the computed result is zero, the function reads the optional record address from offset 0 of the object currently referenced by the slot.

If that address is nonzero:

- The record’s first and second 32-bit words are read and passed, respectively, to F-1d060f30d5b5.
- If that call returns, the saved record address is passed to F-feee9156a257.
- This invocation of F-feee9156a257 is annotated as non-returning.

The record address passed to F-feee9156a257 is the one already obtained before F-1d060f30d5b5. It is not reloaded from the object after that dependency returns.

The evidence establishes the order of the two argument values, but not which record word is physically read first during argument evaluation.

Under the non-returning annotation, this branch does not proceed to F-afd06380d3b7, the offset-8 check, or the final slot-clearing write. F-1d060f30d5b5 may itself alter state or fail to return; its behavior is unknown.

If the optional record address is zero, this pair of dependency calls is skipped, and processing continues to the next stage.

### Zero-result path: later cleanup

After the optional-record stage has been bypassed, the function invokes F-afd06380d3b7 with the current contents of the caller’s slot as an explicit argument.

This differs from its use on the nonzero-result path, where no explicit argument is shown. The supplied material does not resolve the dependency’s true interface.

After F-afd06380d3b7 returns, the function rereads the caller’s slot and retains that object address for the remaining cleanup:

- If the reread value is zero, it skips the remaining object accesses, writes zero to the slot, and returns.
- If it is nonzero, it reads the 32-bit field at offset 8 of that saved object.

There is no guarantee that this saved object is the same object whose offset-4 field was decremented. F-afd06380d3b7 may have changed the slot.

If the offset-8 value is nonzero:

- F-a600efb7b035 is invoked with no explicit arguments shown.
- If it returns, the offset-8 field is read again from the saved object.
- That newly read value is passed to F-feee9156a257.
- This invocation is annotated as non-returning.

The second offset-8 read matters. The value passed to F-feee9156a257 is not necessarily the nonzero value that caused entry into this branch. F-a600efb7b035 may change the field. The second value is not checked again, so even zero can be passed if the dependency changes the field to zero.

The object address used for this second field read is the saved address from before F-a600efb7b035. A change to the caller’s slot alone does not redirect that read.

If the first offset-8 read is zero, the function instead passes the saved object address directly to F-feee9156a257. That invocation is also annotated as non-returning.

Under these annotations, neither branch reaches the final slot write or a normal return.

## Side effects

### Direct writes

The function directly performs at most these writes:

- A 32-bit decrement result at object offset 4.
- Zero to the caller’s slot, if execution reaches normal completion after the initial nonzero gate.

It does not explicitly clear the record address, clear the offset-8 field, or restore the decremented field.

The slot-clear is not an unconditional cleanup guarantee. Annotated non-returning calls can prevent it.

### Dependency effects and sequencing

The dependency calls occur in a meaningful order:

- F-c69e18c9c6b7 precedes the decrement.
- The decrement write precedes the zero comparison’s selected cleanup work.
- On the optional-record branch, F-1d060f30d5b5 precedes F-feee9156a257.
- On later cleanup, F-afd06380d3b7 precedes the slot reread and offset-8 inspection.
- F-a600efb7b035 precedes the second offset-8 read and its F-feee9156a257 call.

No synchronization primitive, atomic operation, memory barrier, or ownership transfer can be established solely from these call positions. Their arrangement may suggest synchronization or release sequencing, but those meanings remain inferred.

### Errors and abnormal control flow

There is no explicit recovery from invalid addresses, inaccessible fields, dependency failures, or arithmetic overflow.

A dependency that does not return prevents all subsequent actions on that path. The function provides no visible rollback of the decrement.

The supplied non-returning annotations do not identify whether control ends through process termination, an exception, a nonlocal transfer, or another mechanism.

## Dependencies

| Opaque ID | Explicit inputs shown | Use in this function | Unresolved contract |
|---|---|---|---|
| F-c69e18c9c6b7 | None | Called after the initial nonzero check, before decrementing | State changes, synchronization, return behavior |
| F-afd06380d3b7 | None on the nonzero-result path; current slot value on the zero-result path | Called before final slot clearing or later cleanup | Actual signature, state changes, synchronization |
| F-1d060f30d5b5 | Two 32-bit words from the optional record | Called before submitting that record to F-feee9156a257 | Value meanings, ownership, memory mutation, return behavior |
| F-a600efb7b035 | None | Called after a nonzero offset-8 check and before rereading that field | State changes, memory lifetime effects, return behavior |
| F-feee9156a257 | One 32-bit address-like value | Receives the optional record, the reread offset-8 value, or the saved object | Actual operation and ownership effects; each call is annotated as non-returning |

“No explicit inputs shown” is not proof that a dependency has no hidden inputs or cannot affect reachable memory.

## Acceptance examples

These examples are conditional static expectations, not runtime test results.

1. **Initially empty slot.**  
   Given a valid slot containing zero, the function returns without dependency calls or writes. Storage for an object is unnecessary on this path.

2. **Ordinary remaining-reference case.**  
   Given a nonzero slot, an offset-4 value of two after F-c69e18c9c6b7 returns, and dependencies that return without altering relevant storage, the field becomes one. F-afd06380d3b7 is called with no explicit argument shown, the slot becomes zero, and the function returns.

3. **Zero field does not trigger zero-result cleanup.**  
   Given the same stability conditions but an offset-4 value of zero, the field becomes negative one. The function takes the nonzero-result path rather than inspecting the optional record.

4. **Optional-record cleanup.**  
   Given an offset-4 value of one and a nonzero optional-record address, the field becomes zero. The record’s two words are passed to F-1d060f30d5b5, followed, if it returns, by the saved record address being passed to F-feee9156a257. Under the non-returning annotation, no subsequent slot-clear occurs.

5. **Slot cleared by the intermediate dependency.**  
   Given an offset-4 value of one and a zero optional-record address, F-afd06380d3b7 receives the current slot value. If it returns after setting the slot to zero, the function performs no offset-8 access, writes zero to the slot, and returns.

6. **Direct object submission.**  
   Under the preceding zero-result conditions, if F-afd06380d3b7 leaves a valid nonzero object in the slot and that object’s offset-8 field is zero, the saved object address is passed to F-feee9156a257. Under its annotation, control does not return normally.

7. **Offset-8 mutation is observed.**  
   If the first offset-8 read is nonzero and F-a600efb7b035 changes that field before returning, F-feee9156a257 receives the changed value. There is no second nonzero check.

8. **Initial dependency changes the target.**  
   If F-c69e18c9c6b7 changes the slot to a different valid object, the subsequent decrement targets the object then referenced, not necessarily the initial object. If it changes the slot to zero, there is no explicit guard preventing the ensuing offset-4 access.

## Uncertainties

The following are **documented_only**:

- The actual calling convention and complete dependency signatures.
- Whether the non-returning annotations accurately describe runtime behavior.
- What would happen if an annotated non-returning call unexpectedly returned.
- Whether dependency calls acquire or release synchronization.
- Whether any dependency frees storage or transfers ownership.
- Whether callbacks or other threads can mutate the slot or objects.
- Whether overlapping slot and object storage is permitted.
- The runtime treatment of signed 32-bit overflow.
- Complete object sizes, allocation origins, alignment, and field meanings.
- Any guarantees of exception safety, thread safety, or reentrancy.

Reference counting, unlocking, and deallocation are plausible interpretations, not established dependency contracts.

## Evidence

This contract is based only on the supplied static function representation and dependency-ID mapping.

Established observations include the zero-slot early return; the 32-bit decrement and equality test; accesses at byte offsets 0, 4, and 8; the two-word record forwarding; dependency call order; slot and field rereads; explicit slot-clearing paths; and the non-returning annotations at each F-feee9156a257 call site.

No dependency bodies, runtime observations, or external ownership specifications were supplied. No runtime tests have been performed.
