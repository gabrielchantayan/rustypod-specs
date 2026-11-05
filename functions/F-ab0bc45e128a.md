<!-- Trusted provenance: F-ab0bc45e128a; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-ab0bc45e128a

## Purpose

F-ab0bc45e128a conditionally configures a context using two binary options and a third option that controls a context byte. Depending on context state, it may also invoke a dependency with a 32-bit value obtained from that context.

The established behavior has two distinct paths:

- If either binary option is outside the accepted range, return the original context argument unchanged, without invoking dependencies or accessing the described global and context storage.
- Otherwise, optionally obtain a context, publish the binary options into shared storage, invoke two dependencies, and inspect context state to determine subsequent writes and the return value.

The interpretation as “configuration” is inferred intent. The actual meanings of the shared flags, context fields, dependency operations, and numeric results are not established.

## Interface

### Inputs

The function accepts four arguments:

| Argument | Representation | Established role |
|---|---|---|
| Context | Integer carrying an address-like value | Returned unchanged on rejection; otherwise used as the base address for context accesses. Zero requests a replacement through F-bb50a072e902. |
| First binary option | Unsigned 32-bit value | Accepted only when zero or one; controls a shared byte at offset +2. |
| Second binary option | Unsigned 32-bit value | Accepted only when zero or one; controls a shared byte at offset +1 and an argument to F-a1497313173e. |
| Control option | Unsigned 32-bit value | Produces a byte equal to one for zero and zero for every other value. It is not subject to the binary-option rejection rule. |

The supplied representation uses integer-shaped context values and address arithmetic. A 32-bit address/return ABI is consistent with that representation, but the original declaration and ABI are not independently established. The two option comparisons are unsigned. The context member at offset +0x94 is explicitly read as a four-byte value.

### Output

The function has an integer-shaped return value:

| Condition | Result |
|---|---|
| Either binary option exceeds one | Original context argument |
| Accepted options; context byte at +0x90 is zero when inspected | 6 |
| Accepted options; +0x90 is nonzero; +0x98 is zero when inspected | 0 |
| Accepted options; +0x90 and +0x98 are nonzero; F-7e415cb3218c returns zero | 0 |
| Accepted options; +0x90 and +0x98 are nonzero; F-7e415cb3218c returns nonzero | 7 |

These are control-flow results, not established error-code names. The rejection result is not a dedicated status: it can numerically equal 0, 6, or 7 if the original context argument has that value.

### Storage layout

The function directly uses the following storage:

| Storage | Offset | Width | Direct use |
|---|---:|---:|---|
| Shared block reached through a global reference | +1 | One byte | Write normalized second option; later read as a dependency argument |
| Shared block reached through a global reference | +2 | One byte | Write normalized first option |
| Context | +0x90 | One byte | Read as a zero/nonzero condition |
| Context | +0x94 | Four bytes | Conditionally read as an argument to F-7e415cb3218c |
| Context | +0x98 | One byte | Conditionally read as a zero/nonzero condition |
| Context | +0x9D | One byte | Conditionally write the derived control byte |
| Context | +0x9E | One byte | Conditionally write one, and possibly later zero |

No larger structure layout, field type, alignment guarantee, or allocation size is established by these accesses alone.

## Behavior

### 1. Reject unsupported binary options

Validation occurs before context acquisition, shared-state access, or dependency calls.

The first binary option is checked first. If it exceeds one, the function immediately returns the original context argument; the second option does not affect that decision.

If the first option is zero or one, the second option determines acceptance. A second option exceeding one also causes immediate return of the original context argument.

Thus, proceeding requires both binary options to belong to the set {0, 1}. Large unsigned values are rejected rather than truncated, treated as signed negatives, or normalized to true.

The control option does not participate in rejection.

### 2. Select the context

For accepted binary options:

- A nonzero context argument is retained.
- A zero context argument causes one call to F-bb50a072e902 with the integer argument one. Its result becomes the context used by subsequent calls and direct accesses.

There is no explicit check of the replacement result. In particular, a zero result does not cause a defined early failure return. Subsequent operations still receive or access that result.

The function does not explicitly release either a supplied context or a replacement context.

### 3. Publish the binary options

After selecting the context, the function captures the shared-block reference for subsequent use.

It performs these direct writes in order:

1. Store the truth value of the first binary option at shared offset +2.
2. Store the truth value of the second binary option at offset +1 through the captured shared-block reference.

Each write is one byte and represents zero or one. Because validation has already succeeded, these values also match the accepted numeric option values.

The supplied representation separately obtains the address used for the first store from the global reference after capturing the reference used for the second store and later read. With a stable global reference, these accesses concern the same block. Behavior under concurrent replacement of that reference is not established.

### 4. Invoke the first two context dependencies

The function calls F-88a504134c0a with the selected context. Its return value is not used.

After that call returns, the function reads one byte at offset +1 through the previously captured shared-block reference. It then calls F-a1497313173e with:

- The selected context.
- The byte just read.

The byte passed to F-a1497313173e is therefore not necessarily the original second option. F-88a504134c0a, or another permitted writer, may have changed the stored byte before the read. The read value is not explicitly normalized again.

A dependency’s replacement of the global shared-block reference does not redirect this later read: it uses the captured reference. Replacement alone also does not establish that the old block remains valid.

The return value of F-a1497313173e is not used. The effects of either dependency remain relevant to all later reads.

### 5. Derive the control byte

After the first two context dependencies, the function derives a one-byte value from the control option:

| Control option | Derived byte |
|---:|---:|
| 0 | 1 |
| 1 | 0 |
| Any unsigned value greater than 1 | 0 |

Although the represented computation includes a character-width conversion and subtraction, the explicit greater-than-one case forces the final value to zero for every larger unsigned input. The externally relevant result is therefore an exact zero test, not parity, low-byte truncation, or subtraction modulo 256.

### 6. Inspect the context condition

The function reads the context byte at +0x90 after both context dependencies have returned.

If that byte is zero, the function returns 6 immediately. On this branch, it performs no direct write to +0x9D or +0x9E and makes no call to F-7e415cb3218c.

This does not undo the earlier shared writes or dependency effects.

If the byte is nonzero, the function proceeds without later rechecking it.

### 7. Update context bytes

On the nonzero +0x90 branch, the function performs these writes in order:

1. Write one to context offset +0x9E.
2. Write the derived control byte to context offset +0x9D.

It then reads the context byte at +0x98.

If +0x98 is zero, the function returns zero. Its last direct write to +0x9E on this path is one.

The order is significant: the +0x98 read occurs after both context writes. If the underlying storage aliases in an unusual layout or can be changed externally, the decision depends on the value observed at that point.

### 8. Conditionally invoke the final dependency

If the observed +0x98 byte is nonzero, the function reads four bytes at context offset +0x94 and passes that value to F-7e415cb3218c.

It distinguishes only a zero return from a nonzero return:

- Nonzero dependency result: return 7, with no further direct context write.
- Zero dependency result: write zero to context offset +0x9E, then return zero.

The final dependency’s return value is not propagated directly.

A nonzero result does not prove that +0x9E remains one: the dependency may itself have modified that byte. A zero result does establish a subsequent direct zero write by this function, assuming the context storage remains valid.

## Side effects

The direct effects are ordered as described above. In particular:

- Rejected binary options produce no described shared or context access.
- Accepted options update shared bytes before either context dependency.
- A return of 6 can occur after shared-state mutation and both context dependency calls.
- Context +0x9E is written before +0x9D.
- The +0x98 condition is read after those writes.
- The four-byte member at +0x94 is read only when the observed +0x98 byte is nonzero.
- A zero result from the final dependency causes a later clearing write to +0x9E.

There is no rollback, explicit deallocation, or explicit ownership transfer in the function body. These absences do not constrain what the dependencies may do.

Normal completion requires the storage used at each access to remain valid and appropriately accessible. That includes the captured shared block after F-88a504134c0a returns and the context after each dependency call. Dependencies must return normally for later operations to occur.

No synchronization, atomicity, exception recovery, invalid-address recovery, or acquisition-failure recovery is established. Observations involving concurrently modified storage are documented_only.

## Dependencies

| Opaque ID | Invocation and result use | Unestablished behavior |
|---|---|---|
| F-bb50a072e902 | Called once with one only when accepted options accompany a zero context. Result replaces the context. | Whether it allocates, returns existing storage, transfers ownership, or can fail |
| F-88a504134c0a | Called with the selected context after shared writes. Return ignored. | Context mutations, shared-state mutations, lifetime effects, and other side effects |
| F-a1497313173e | Called with the selected context and the byte read from captured shared offset +1 after the preceding call. Return ignored. | Meaning of the byte and effects on context, shared storage, or ownership |
| F-7e415cb3218c | Called only after nonzero observations at context +0x90 and +0x98. Receives the four-byte member at +0x94. Zero/nonzero result controls clearing and return. | Meaning of its argument, resource effects, memory effects, and meaning of nonzero results |

No dependency is assumed to preserve globals, preserve context contents, or retain versus release memory. Any stronger dependency contract is documented_only until independently established.

## Acceptance examples

These are conditional acceptance scenarios derived from static evidence, not executed tests. Except where stated otherwise, they assume stable, valid, non-overlapping shared and context storage and normally returning dependencies.

1. **Invalid first option bypasses acquisition.**  
   With context zero and first binary option two, the result is zero—the original context argument. F-bb50a072e902 is not called, regardless of the other options.

2. **Invalid second option preserves the original result value.**  
   With a valid first option, second option three, and a nonzero context, the original context argument is returned. No shared flag or context member is accessed by this function.

3. **A missing context is replaced before configuration.**  
   With both binary options accepted and context zero, F-bb50a072e902 receives one. If it returns a valid context, all subsequent context calls and accesses use that result. No explicit failure branch is expected if it instead returns zero.

4. **The second dependency observes a post-call shared byte.**  
   With second option zero, shared offset +1 is initially written as zero. If F-88a504134c0a changes that byte to five and leaves the block valid, F-a1497313173e receives five, not zero or one.

5. **A changed global reference does not redirect the captured read.**  
   If F-88a504134c0a replaces the global shared-block reference but leaves the old block valid, the byte passed to F-a1497313173e comes from offset +1 in the old, captured block.

6. **The zero +0x90 branch retains earlier effects.**  
   If +0x90 is zero after F-a1497313173e, the function returns 6. Shared writes and the two preceding context calls have already occurred. This function does not directly write +0x9D or +0x9E on that branch.

7. **No final call when +0x98 is zero.**  
   If +0x90 is nonzero, +0x98 is zero, and the control option is zero, the function writes one to +0x9E, then one to +0x9D, and returns zero without invoking F-7e415cb3218c.

8. **All nonzero control values produce zero.**  
   On a path that writes +0x9D, control values one, two, 256, and the maximum unsigned 32-bit value each cause a zero write to that byte.

9. **Final dependency success clears the indicator afterward.**  
   With nonzero +0x90 and +0x98, F-7e415cb3218c receives the four-byte value read at +0x94. If it returns zero and leaves the context writable, the function subsequently writes zero to +0x9E and returns zero.

10. **Final dependency failure maps to seven.**  
    Under the same branch conditions, any nonzero dependency result causes return 7. There is no subsequent direct clearing write, but the dependency’s own mutations remain possible.

## Uncertainties

The following are documented_only:

- Original ABI details, signedness of the context declaration, and definitive pointer width.
- Semantic names for the shared bytes and context members.
- Whether obtaining a context entails allocation, reference acquisition, or ownership transfer.
- Whether any dependency can invalidate storage, change global references, or modify the inspected context members.
- The interpretation and ownership of the four-byte value at +0x94.
- Required alignment, complete object sizes, and permissible aliasing.
- Behavior after invalid memory access, nonreturning calls, or concurrent mutation.
- Whether numeric results 0, 6, and 7 have named meanings elsewhere.

Inferred intent does not strengthen these unknowns into guarantees.

## Evidence

This contract is based solely on the supplied static function representation and the supplied opaque dependency mapping.

Established evidence covers branch conditions, direct access widths and offsets, argument construction, dependency-call order, direct-write order, and return selection. Context acquisition and configuration terminology are interpretations, not proven resource semantics.

No dependency implementation, external caller contract, original type declaration, or runtime observation was available. No runtime tests have been performed.
