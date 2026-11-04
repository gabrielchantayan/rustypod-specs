<!-- Trusted provenance: F-9d3acb3a89b1; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-9d3acb3a89b1

## Purpose

F-9d3acb3a89b1 conditionally notifies an associated object, resolves or validates a signed index, and, for an accepted index other than −1, invokes an ordered sequence of operations on the supplied object.

The established behavior is procedural. It does not establish that elements are deleted, that storage is released, or that any particular container invariant is maintained. A plausible intent is to operate on an indexed item and any following range, but the meanings of the invoked operations remain unknown.

A special input value, 2,147,483,647, resolves to one less than a positive object field without invoking the normal validation dependency. A validation result of zero causes an immediate return of −1. Otherwise, the function returns the resolved or original index after any required calls complete.

## Interface

### Inputs

The function receives:

- An object address.
- A signed 32-bit index.

The object is represented as an addressable sequence of 32-bit words. The following portions are used directly:

| Object-relative location | Established use |
|---|---|
| Offset 0 | Holds a dispatch-table address used for two indirect calls. |
| Offset 4 | Holds a signed 32-bit value used as a count or upper-bound field. |
| Offset 40 | Holds a 32-bit word whose conversion to an 8-bit character controls optional preliminary processing. |

“Count” below is a descriptive label for the field at offset 4, not a guarantee that it accurately describes allocated storage.

The flag test concerns the 8-bit result of converting the word at offset 40, rather than whether the entire 32-bit word is nonzero. This distinction matters when the word is nonzero but its low eight value bits are zero. The representation does not establish byte order sufficiently to identify this test with a particular byte-addressed load.

### Output

The function returns a signed 32-bit integer:

- −1 when the validation dependency returns zero.
- Otherwise, the effective index used for subsequent processing.

Consequently, a return of −1 is not sufficient to distinguish validation failure from successful validation of an input index of −1.

### Preconditions

There is no explicit null check for the object address. The object must support the reads required by the executed path.

Additional path-dependent requirements include:

- If preliminary processing returns a non-null associated object, that object must support a dispatch-table lookup and the resulting indirect call.
- If the effective index is not −1, the main object must support dispatch-table lookup at byte displacement 188.
- If the following-range condition is true, the main object must also support dispatch-table lookup at byte displacement 184.
- Any invoked dependency must accept the arguments and calling convention actually used.

These are requirements for meaningful execution, not validations performed by this function. Invalid addresses, incompatible object layouts, invalid indirect targets, or dependency failures have no locally established recovery behavior.

## Behavior

### 1. Optional preliminary processing

The function first examines the character-converted word at object offset 40.

If that value is zero, it skips this phase entirely.

If it is nonzero, the function calls F-8966abe1b944 with:

1. The supplied object address.
2. The original input index.

This happens before interpreting the special maximum-index value and before calling the validation dependency. F-8966abe1b944 therefore receives the original index, including 2,147,483,647 or −1 when either is supplied.

If F-8966abe1b944 returns a null address, no associated-object indirect call occurs.

If it returns a non-null address:

1. The function reads a dispatch-table address from the beginning of the returned object.
2. It obtains an indirect target from byte displacement 4 in that table.
3. It invokes that target.

The recovered representation supplies no explicit arguments to this indirect call. Its actual calling-convention requirements, implicit arguments, and effects are undocumented. In particular, the evidence does not establish that it is a destructor or that it releases the returned object.

This preliminary phase is not rolled back if later validation fails.

### 2. Resolve the special value or validate the input

After preliminary processing completes, the function tests the index.

If the input index equals 2,147,483,647 and the current count field is strictly positive, the function:

- Replaces the effective index with the count field minus one.
- Does not call F-402eef64af83.

The count used here is observed after the preliminary calls. The function does not establish that those calls preserve it.

For every other case, it calls F-402eef64af83 with the object address and the unchanged input index. This includes:

- Ordinary nonnegative indices.
- Negative indices.
- −1.
- 2,147,483,647 when the count is zero or negative.

If F-402eef64af83 returns zero, the function immediately returns −1. No subsequent range, marker, or final dependency call occurs.

Any nonzero result permits processing to continue. Negative nonzero results are accepted just as positive nonzero results are. This function does not impose an additional local bounds check on the accepted index.

### 3. Handle the effective index

If the effective index is −1, the function performs no further calls and returns −1.

Otherwise, it evaluates whether the effective index plus one is less than the count field.

When that condition is true, it invokes the main object’s indirect target selected through dispatch-table byte displacement 184. The logical arguments visible in the recovered representation are:

1. The main object address.
2. The effective index plus one.
3. The count field minus one.
4. A 32-bit value with all bits set.

Additional machine-state-derived values appear in the recovered call representation. Their values, argument status, and meaning are not established. They must not be treated as initialized, stable public inputs.

The comparison is strict. For ordinary nonnegative indices and a coherent positive count, the range call is omitted when the effective index denotes the last position.

The function then invokes the main object’s indirect target selected through dispatch-table byte displacement 188, regardless of whether the preceding range call occurred. The visible arguments are:

1. The main object address.
2. A 32-bit value with all bits set.

Finally, it calls F-d6032bc0598f with:

1. The main object address.
2. The effective index.

It then returns that effective index.

### 4. Required ordering

For a path that reaches every operation, the observable call order is:

1. F-8966abe1b944.
2. The associated-object indirect call.
3. F-402eef64af83, unless special-value resolution bypasses it.
4. The main-object range indirect call.
5. The main-object marker indirect call.
6. F-d6032bc0598f.

Each optional call is omitted only under its stated condition.

The dispatch-table address for the marker call is obtained after the range call. The function does not establish that the range call preserves the object’s dispatch-table field. It is therefore not valid to specify that both main-object calls necessarily use the same table.

Likewise, object fields are not contractually frozen across dependency calls. Validation can affect the count consulted afterward, and preliminary processing can affect the count used for special-value resolution.

### 5. Integer boundaries

The index and count are signed 32-bit quantities. Relevant constants are:

- Maximum signed value: 2,147,483,647.
- Skip value and failure result: −1.
- All-bits-set call argument: the 32-bit pattern `0xFFFFFFFF`.

The all-bits-set argument represents −1 if interpreted as signed 32-bit data. Its semantic meaning to either indirect target is unknown.

The following arithmetic requires boundary care:

- Special-value resolution subtracts one from a count observed as positive. With a stable count, the result is representable.
- Subsequent processing computes effective index plus one.
- The range call computes count minus one.

An accepted effective index of 2,147,483,647 reaches an addition that is outside the signed 32-bit range. This can happen if the count is nonpositive and F-402eef64af83 nevertheless accepts the special input. A range-call endpoint can likewise require an unrepresentable subtraction if the observed count is −2,147,483,648.

The supplied representation does not establish a reliable overflow contract for these cases. Neither wrapping arithmetic nor a particular failure outcome is promised.

## Side effects

No explicit assignment to an object field or global variable appears in the function itself. Its direct effects consist of reads, dependency calls, and returning a result.

All called dependencies may have effects not visible here, including modifying reachable state, changing dispatch tables, or affecting memory ownership. No preservation guarantee may be inferred merely because a dependency’s return value is ignored.

The function contains:

- No explicit allocation.
- No explicit memory release.
- No explicit rollback.
- No explicit exception or fault handling.
- No status check on the range, marker, or final dependency calls.

There is no established ownership transfer for the main object, the object returned by F-8966abe1b944, or any indirectly referenced storage.

If a dependency does not return normally, the evidence establishes no guaranteed continuation or return value.

## Dependencies

| Dependency | Established interaction | Unknowns |
|---|---|---|
| F-8966abe1b944 | Receives the main object and original index when the flag test is nonzero; its result is tested for null. | Result ownership, lifetime, lookup semantics, and side effects. |
| F-402eef64af83 | Receives the main object and original index except on the positive-count special-value path; zero rejects, any nonzero value accepts. | Validation criteria and mutations. |
| F-d6032bc0598f | Receives the main object and effective index after the marker call, for effective indices other than −1. | Operation performed, ownership effects, and return semantics. |

Three further dependencies are indirect and have no supplied opaque function IDs:

- The target at byte displacement 4 in the associated object’s dispatch table.
- The target at byte displacement 184 in the main object’s dispatch table.
- The target at byte displacement 188 in the main object’s dispatch table.

These are dispatch locations, not established identities. Different objects or intervening state changes may select different targets.

## Acceptance examples

These examples are conditional static acceptance cases, not runtime observations. Unless specified otherwise, all invoked dependencies return normally and preserve fields relevant to later decisions.

### Ordinary accepted index with a following range

Given:

- A zero preliminary flag.
- Count 5.
- Input index 2.
- A nonzero result from F-402eef64af83.

The required sequence is:

1. Validate index 2.
2. Invoke the range target with logical endpoints 3 and 4 and an all-bits-set marker.
3. Invoke the marker target with an all-bits-set marker.
4. Invoke F-d6032bc0598f with index 2.
5. Return 2.

No call to F-8966abe1b944 occurs.

### Accepted last index

Given count 5, index 4, and successful validation, the range call is omitted because 5 is not less than 5. The marker call and F-d6032bc0598f still occur, and the return value is 4.

### Positive-count special value

Given count 5 and input 2,147,483,647, the effective index becomes 4 without a call to F-402eef64af83. With the count unchanged, the range call is omitted. The marker call and F-d6032bc0598f occur, and the function returns 4.

If preliminary processing changes the count from 5 to 3 before resolution, the corresponding effective index is 2, not 4.

### Rejection after preliminary effects

Given a nonzero flag, a non-null result from F-8966abe1b944, and a zero result from F-402eef64af83:

- The associated-object indirect call occurs before validation.
- The function returns −1.
- No range, marker, or final dependency call occurs.
- The preliminary effects are not undone.

### Accepted −1

Given input −1 and a nonzero validation result, the function returns −1 without invoking the range target, marker target, or F-d6032bc0598f. Optional preliminary processing still occurs first.

### Another negative index

Given count 5, input −2, and successful validation, the function does not locally reject the negative index. It invokes the range target with endpoints −1 and 4, then the marker target, then F-d6032bc0598f with −2, and returns −2.

Whether such arguments are valid for those dependencies is undocumented.

### Zero character value in a nonzero word

If the flag word is nonzero but converts to the zero 8-bit character value, preliminary processing is skipped. Whole-word nonzero status alone does not enable it.

## Uncertainties

The following are **documented_only**:

- The domain-level meaning of the count field and all indirect operations.
- The concrete identities of indirect targets.
- Argument conventions for the associated-object call and unexplained trailing call-state values.
- Whether any dependency preserves globals, object fields, dispatch tables, or memory ownership.
- Whether dependencies validate storage or index bounds.
- Arithmetic behavior where signed 32-bit operations are not representable.
- Concurrency, synchronization, and reentrant mutation guarantees.
- Physical load counts and instruction-level access order beyond the logical dependencies visible in the supplied representation.
- Failure behavior for invalid pointers, invalid call targets, or dependencies that fail to return normally.

The strongest inferred intent is “resolve an index, process a following range if present, then process the selected position.” This interpretation is not an established semantic guarantee.

## Evidence

This contract is based exclusively on the supplied static function representation and opaque dependency mapping.

Established evidence includes the visible branch conditions, constants, object-relative field positions, dispatch-table displacements, dependency arguments, call ordering, and return expressions.

No dependency implementations, runtime traces, type definitions, ownership documentation, or execution results were supplied. No runtime tests have been performed.
