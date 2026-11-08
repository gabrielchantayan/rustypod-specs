<!-- Trusted provenance: F-44f7ecb688eb; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-44f7ecb688eb

## Purpose

F-44f7ecb688eb conditionally constructs an object from an input record and publishes a reference to it through a caller-supplied output slot.

The construction path requires both a nonzero result from a preliminary dependency and a non-null pointer to a nonzero first byte in the input record. Otherwise, the function delegates output initialization to a separate dependency.

The visible field updates are consistent with reference-counted ownership, but that interpretation is inferred intent. The established contract is the ordering of calls, pointer accesses, counter updates, and return values described below. Allocation, construction, synchronization, and destruction semantics remain conditional on dependencies whose implementations are unavailable.

## Interface

### Inputs

The function receives four arguments:

| Argument | Representation | Visible use |
|---|---|---|
| Output slot | Pointer to writable 32-bit storage | Receives a control-record pointer on the construction path; passed to an output-initialization dependency otherwise. |
| Input record | Pointer to a record addressed in 32-bit words | Passed to the preliminary dependency and subsequently inspected at specific offsets. |
| Auxiliary value A | 32-bit value | Passed to the preliminary dependency and initially stored in temporary storage. |
| Auxiliary value B | 32-bit value | Passed to the preliminary dependency and initially stored in adjacent temporary storage. |

Pointers and record words are represented as 32-bit quantities in the supplied evidence. The function is represented as returning 64 bits.

The output slot and input record are not checked for null before use. A null input record may still reach the fallback path if the preliminary dependency returns zero without requiring it to be valid. A nonzero preliminary result leads to direct input-record access.

### Input-record layout

The following offsets are established by the visible accesses:

| Byte offset | Width | Meaning established by use |
|---|---:|---|
| 20 | Address of a field | Supplied as the second argument to F-b9acea4b06ad. Its full readable extent is unknown. |
| 24 | 32 bits | Interpreted as a pointer to a byte sequence. |
| 28 | 32 bits | Supplied as the third argument to F-3475b4980949. |
| 36 | 32 bits read, then narrowed | Converted to an 8-bit character value and promoted for a constructor argument. |

Only the first byte of the sequence referenced at offset 24 is examined directly. The function does not directly establish a sequence length or search for a terminator.

### Control-record layout

On the construction path, a temporary pointer is passed by address to F-daedc77d3db6. Subsequent operations treat its resulting value as pointing to a record with these fields:

| Byte offset | Width | Visible role |
|---|---:|---|
| 0 | 32 bits | Pointer to another object. |
| 4 | 32 bits | Counter incremented during publication and decremented during temporary-reference cleanup. |
| 8 | 32 bits | Controls calls surrounding the increment and participates in terminal cleanup. |

These roles do not establish a complete record layout or a general-purpose ownership protocol.

### Return value

Two return forms are visible:

- A normally completed construction path returns the 64-bit value zero.
- The fallback path returns a 64-bit concatenation whose upper 32 bits are the temporary record-pointer value and whose lower 32 bits are the output-slot address.

On the fallback path, the temporary record pointer has been initialized to the input-record pointer and has not been explicitly reassigned. Thus, under ordinary non-aliasing execution, the returned upper half is the input-record address and the lower half is the output-slot address. This is not a Boolean success result.

The externally intended interpretation of this 64-bit return is unknown. The supplied representation may expose an ABI-level result rather than the original interface abstraction.

## Behavior

### 1. Preliminary decision

Before calling any dependency, the function prepares local storage containing:

- The input-record pointer.
- Auxiliary value A.
- Auxiliary value B.

It calls F-1ddc1da0c68f with the input-record pointer and both auxiliary values.

If the result is zero, the function takes the fallback path without directly reading the input record’s byte-sequence pointer.

If the result is nonzero, the function reads the 32-bit pointer at input-record offset 24. When that pointer is non-null, it reads one byte through it. Construction proceeds only if both the pointer and that byte are nonzero.

The record inspection occurs after the preliminary call. Consequently, the inspected state is whatever is present after that call returns, not necessarily the state present on entry.

### 2. Fallback path

The fallback path is selected when any of these conditions holds:

- F-1ddc1da0c68f returns zero.
- The pointer at input-record offset 24 is null.
- That pointer is non-null, but its first byte is zero.

The function calls F-daedc77d3db6 with:

1. The caller’s output-slot address.
2. Zero.
3. Zero.

It then returns the concatenated value described under **Interface**.

There is no direct assignment of zero to the output slot in this path. Whether the dependency clears the slot, replaces an existing reference, releases an old object, or performs another operation is unknown.

### 3. Construction and temporary initialization

Once the construction condition is satisfied, the function performs these operations in order:

1. Calls F-8157a180d901 with the value 12.
2. Calls F-b9acea4b06ad with the address of the temporary storage initially holding auxiliary value A and the address of input-record offset 20.
3. Calls F-e5099eaf0347 with:
   - The result of F-8157a180d901.
   - The temporary-storage address used in the preceding call.
   - The constant 1.
   - A character-width conversion of the word read from input-record offset 36.
4. Calls F-daedc77d3db6 with the address of the temporary record-pointer slot, the constructor result, and zero.
5. Calls F-289ec4311c64 with the temporary-storage address.

The constant 12 is established. Its interpretation as an allocation size is inferred from use, not proven by the dependency name or body.

The temporary storage begins with the two auxiliary values, but F-b9acea4b06ad may overwrite it. No preservation of either original auxiliary value is assumed afterward.

The offset-36 value is narrowed to a character-sized value before promotion. The precise promoted result for values whose low byte is at least 128 depends on the target’s plain-character signedness, which the supplied evidence does not independently establish.

### 4. Applying the input’s offset-28 value

After temporary finalization, the function derives a pointer as follows:

- If the temporary record pointer is non-null, it reads that record’s offset-0 word and treats the result as another pointer.
- If the temporary record pointer is null, it uses a null intermediate pointer.

It then unconditionally reads a 32-bit word through that intermediate pointer to form the first argument to F-3475b4980949.

The call also receives:

- A second argument represented as a register-carried value with no explicit assignment in the supplied body.
- The current 32-bit value at input-record offset 28.
- Zero.

Consequences:

- A null temporary record pointer is not safely handled at this stage.
- A non-null record whose offset-0 pointer is null also leads to an invalid dereference before the call.
- There is no visible constructor-failure check that bypasses these accesses.
- The second argument cannot be specified as a stable value, a duplicate of another argument, or a preserved global.

The relative evaluation order of the argument-forming reads is not established. Both the object-related dereference and the input-record offset-28 read precede entry into F-3475b4980949.

### 5. Publishing the reference

After F-3475b4980949 returns, the function captures the current temporary record pointer and writes that pointer into the caller’s output slot.

For a non-null captured pointer:

1. It reads the word at control-record offset 8.
2. If that word is nonzero, it calls F-338d16f9ce72.
3. It increments the 32-bit counter at offset 4.
4. It reads offset 8 again.
5. If that word is nonzero, it calls F-1333331b2da1.

The two offset-8 observations are separate reads. A specification must not collapse them into one cached condition: intervening calls may have effects not established here.

The output write occurs before the counter increment. Publication is therefore not visibly an “increment first, publish second” operation.

The captured pointer is used for this increment sequence. It must not be assumed to track later changes to the temporary pointer slot.

After the increment sequence, the function checks the current temporary pointer slot. If it is null, the function returns zero without entering the subsequent decrement sequence. This check does not make the earlier unconditional object dereference safe.

### 6. Releasing the temporary reference

When the current temporary record pointer is non-null, the function:

1. Calls F-25197b74a768.
2. Reads the counter at offset 4 through the current temporary record pointer.
3. Writes that counter minus one back to offset 4.
4. Branches on whether the decremented result is zero.

If the result is nonzero, it calls F-8a8c41f7d667 and returns zero.

If the result is zero:

- When the record’s offset-0 word is nonzero, it calls F-78645499736b and then F-feee9156a257. The supplied representation marks the latter call as non-returning.
- When the offset-0 word is zero, it calls F-8a8c41f7d667 with the current temporary record pointer, then reloads that pointer:
  - If it is null, the function reaches the zero return.
  - If it is non-null and its offset-8 word is zero, it calls F-feee9156a257 with the record pointer; this call is marked non-returning.
  - Otherwise, it calls F-a600efb7b035 and then F-feee9156a257 with the offset-8 word read through the captured record pointer; this final call is also marked non-returning.

The evidence does not establish valid continuation behavior if a call marked non-returning unexpectedly returns.

### Arithmetic

The counter accesses and updates are 32-bit operations. No overflow, underflow, saturation, or invalid-count check is visible.

For machine-word interpretation, incrementing the maximum 32-bit unsigned pattern wraps to zero, and decrementing zero produces the maximum 32-bit unsigned pattern. Whether all such states are valid under the original language and ownership protocol is unknown. The branch after decrement tests the resulting value, not the original value.

A simple “one increment cancels one decrement” guarantee requires additional assumptions: the same record remains selected, no intervening dependency changes the counter, and no relevant alias changes its storage.

## Side effects

Established direct effects include:

- One output-slot write on the construction path.
- A counter increment for a non-null captured published record.
- A later counter decrement when the current temporary record pointer remains non-null.
- Reads from the input record, a pointed-to byte, the control record, and a further pointed-to object word.

Dependencies may introduce additional writes, allocation, release, synchronization, or external effects. None of those effects is assumed absent.

The output slot’s previous contents are not visibly inspected or released before the direct construction-path assignment. Callers must not assume this path safely replaces a live owned reference.

No non-aliasing requirement is stated by the evidence. Aliasing can affect behavior: an output slot overlapping a control field changes that field before subsequent counter operations. The function contains no explicit overlap defense.

The temporary storage and temporary record-pointer slot are passed by address to dependencies. Whether any dependency retains either address is unknown. Valid lifetime and non-retention requirements cannot be established from this body alone.

## Dependencies

| Opaque ID | Established interaction |
|---|---|
| F-1ddc1da0c68f | Receives the input record and both auxiliary values; zero selects fallback. |
| F-8157a180d901 | Receives 12; its result becomes the first constructor argument. |
| F-b9acea4b06ad | Receives temporary storage and input-record offset 20. |
| F-e5099eaf0347 | Receives the resource result, temporary storage, 1, and the narrowed character value; returns the value supplied to temporary-record initialization. |
| F-daedc77d3db6 | Used both to initialize the temporary record pointer and to process the caller’s output slot on fallback. |
| F-289ec4311c64 | Receives temporary storage after construction and temporary-record initialization. |
| F-3475b4980949 | Receives a dereferenced object word, an unresolved register-carried argument, input-record offset 28, and zero. |
| F-338d16f9ce72 | Conditionally called before increment when control-record offset 8 is nonzero. No explicit arguments are shown. |
| F-1333331b2da1 | Conditionally called after increment under a fresh offset-8 test. No explicit arguments are shown. |
| F-25197b74a768 | Called before the cleanup decrement. No explicit arguments are shown. |
| F-8a8c41f7d667 | Called after a nonterminal decrement, or on the zero-count/null-object branch. The latter call explicitly supplies the record pointer. |
| F-78645499736b | Called when decrement reaches zero and the object word is nonzero. No explicit arguments are shown. |
| F-feee9156a257 | Appears in terminal cleanup with different visible argument forms; marked non-returning. |
| F-a600efb7b035 | Called on the zero-count/null-object branch when the remaining record’s offset-8 word is nonzero. No explicit arguments are shown. |

An absent explicit argument list does not prove that a dependency receives no ABI-level context.

## Acceptance examples

These are conditional static acceptance scenarios, not executed tests.

1. **Preliminary rejection.** If F-1ddc1da0c68f returns zero, the function calls F-daedc77d3db6 with the output slot and two zeros. It does not directly read the byte-sequence pointer. Under ordinary non-aliasing execution, the return contains the input-record address in its upper half and the output-slot address in its lower half.

2. **Null or empty byte sequence.** If preliminary acceptance is nonzero but offset 24 is null, fallback occurs without a byte dereference. If it points to a zero byte, exactly that initial byte is sufficient to select fallback.

3. **Nonzero first byte.** If preliminary acceptance is nonzero and the pointed-to byte is nonzero, construction begins with the call carrying 12. No direct scan of subsequent bytes is required by this function.

4. **Stable shared record.** Suppose dependencies establish a valid record and object pointer, preserve that record selection, leave its counter initially at 3, and do not otherwise change it. Publication writes the record pointer, the counter becomes 4, cleanup restores it to 3, the nonterminal cleanup dependency runs, and the function returns zero.

5. **Independent synchronization conditions.** If offset 8 is nonzero before increment but becomes zero before the second test, the pre-increment conditional dependency is called and the post-increment conditional dependency is skipped.

6. **Null constructed record.** If temporary-record initialization leaves a null pointer and nothing changes it before object access, the path reaches a null dereference. A successful empty result is not established.

7. **Counter reaches zero.** If cleanup observes a counter of 1, it writes zero and enters terminal handling. A nonzero object word selects F-78645499736b followed by the marked non-returning dependency.

8. **Counter starts at zero during cleanup.** Under 32-bit machine arithmetic, decrement produces all one bits, so the zero-result branch is not selected.

## Uncertainties

The following are **documented_only**:

- Dependency implementations, complete calling conventions, and implicit register inputs.
- Allocation failure behavior and ownership of the value obtained using 12.
- Whether output initialization releases an existing reference.
- The meaning of the unresolved second argument to F-3475b4980949.
- Character signedness for low-byte values from 128 through 255.
- Whether the counter protocol is thread-safe, atomic, or protected by the conditional calls.
- Whether dependencies preserve globals, counters, pointer slots, or object fields.
- Whether terminal calls truly cannot return.
- Exception, trap, cancellation, and allocation-failure propagation.
- The intended external meaning of the represented 64-bit return.
- Required readable extents of temporary values and dependency-consumed input fields.

“Allocation,” “construction,” “reference count,” and “synchronization” are explanatory interpretations, not independently established dependency contracts.

## Evidence

This contract is based solely on the supplied static function representation and dependency-ID mapping. Established evidence consists of explicit argument values, field offsets, access widths, branch conditions, call ordering, direct stores, counter arithmetic, and represented returns.

No implementation bodies for dependencies were supplied. No runtime tests, executions, memory traces, or ABI validation have been performed. The acceptance examples are conditional consequences of the visible control flow, not claims of observed execution.
