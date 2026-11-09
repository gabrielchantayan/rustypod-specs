<!-- Trusted provenance: F-bde1b53ec500; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-bde1b53ec500

## Purpose

F-bde1b53ec500 coordinates processing of an input record through a context-associated state object. It obtains an initial quantity and position, resolves a state object, optionally enforces a fresh-state condition, and repeatedly submits the remaining input to a processing dependency.

The processing dependency has three observably different outcomes:

- A zero result causes the current state to be detached and handed to another dependency, followed by advancement of the input position and reduction of the remaining quantity.
- The distinguished result value 37 is accepted only when the dependency’s reported quantity equals the entire quantity remaining for that attempt.
- Every other nonzero result, including 37 with a mismatched reported quantity, enters a nonreturning failure path.

A record flag independently controls a final indirect call whose result becomes the function result.

This is an operational contract based on supplied static evidence. The semantic meanings of the input, state objects, and processing results remain partly unknown. No runtime tests have been performed.

## Interface

### Inputs

The function receives four machine-word arguments:

1. **Context reference:** a reference to a word-addressed object. Its 32-bit word at byte offset 336 is read to locate an associated lookup structure.
2. **Record reference:** a reference to an object containing at least:
   - A dispatch-table reference at byte offset 0.
   - A 32-bit reference at byte offset 4. A 16-bit flag field is read at byte offset 10 within the object referenced by that second word.
3. **Auxiliary word A:** an opaque 32-bit value.
4. **Auxiliary word B:** an opaque 32-bit value.

The auxiliary words are copied into working storage at entry. The supplied body does not interpret them directly.

The evidenced representation uses 32-bit words and references, a 16-bit flag load, and a 64-bit returned value. The apparent use of a reference type for some arithmetic operands does not establish that those operands designate objects.

### Required readable structures

For a normally completing invocation:

- The context reference must support the read at byte offset 336.
- The record must support the initial dependency calls and later flag reads.
- The record’s second word must identify storage readable as a 16-bit value at relative byte offset 10.
- A resolved state object, whenever used, must support access to a dispatch-bearing subobject at byte offset 4.
- Any indirect dispatch actually taken must resolve to a callable target with the evidenced argument convention.

These are access preconditions, not validations performed by the function.

A null record is not safely handled by the visible control flow: dependencies receive it before any null check, and the final flag expression dereferences it before testing whether it is null.

### Result

The function has two normal-result forms:

- If the final record flag requests indirect completion, the function returns the 64-bit result of that indirect call.
- Otherwise, it returns two working 32-bit words combined into a 64-bit value:
  - The upper 32 bits come from a working quantity/output word.
  - The lower 32 bits come from a working reference word.

The exact values of these words depend on whether a processing attempt occurred; details appear below. This combined result must not be assumed to be an ordinary success code or a single pointer.

Certain paths invoke dependencies represented as nonreturning. Those paths do not produce a normal result under the supplied evidence.

## Behavior

### 1. Establish input descriptors and resolve state

The function first initializes three working words from the record reference and the two auxiliary arguments.

It then makes these calls in order:

1. F-b1020422996f, with the record reference. Its result becomes the **remaining quantity**.
2. F-2dd0aac96475, with the record reference. Its result becomes the **current position**.
3. F-407933f268e5, with the record reference. Its result becomes a **lookup key**.
4. F-b37968c9f644, with:
   - The 32-bit value read from context byte offset 336, increased by 4.
   - The lookup key.

The fourth result becomes the **resolved state object**.

The first two results are subsequently used numerically: one as a quantity, the other as a position advanced by reported progress. Their actual application-level units are not established.

The context-derived addition occurs on the loaded word, not on the address of the context field itself.

### 2. Apply the fresh-state flag

The function reads the record’s 16-bit flag field. If bit 1, mask 0x0002, is set:

1. It calls the resolved state object’s indirect operation at dispatch-table byte offset 4, passing the state subobject at object byte offset 4.
2. If that operation returns a nonzero value:
   - F-09ba55c206ac is called with no explicit arguments.
   - F-feee9156a257 is then called on a path represented as nonreturning.
3. If the operation returns zero, the indirect operation at dispatch-table byte offset 8 is called with:
   - The same state subobject.
   - A zero value.

This flag-controlled sequence occurs before the resolved-state null check used to enter the processing loop. Consequently, a null lookup result is not protected from dereference when bit 1 is set.

The two indirect operations are used as state retrieval and state installation operations. Those roles describe their visible use; their internal effects and ownership rules are unknown.

After this sequence, the working reference word is assigned the context reference.

### 3. Decide whether to process

The processing loop is entered only if all three values are nonzero:

- Remaining quantity.
- Current position.
- Resolved state object.

A zero in any of these values skips the loop. Skipping the loop does not skip the final flag read or possible indirect completion call.

### 4. Obtain or create the current processing state

At the start of each attempt, the function retrieves the current state through dispatch-table byte offset 4 on the resolved object’s byte-offset-4 subobject.

If the returned state is nonzero, processing uses it directly.

If it is zero:

1. F-b85dfaa750f8 is called with the resolved state object.
2. If that call returns zero, F-a817d8d58c10 is invoked on a nonreturning failure path.
3. Otherwise, the returned value is installed through the indirect operation at dispatch-table byte offset 8.

There is no visible retry after a zero result from F-b85dfaa750f8.

### 5. Prepare and submit an attempt

Once a nonzero processing state is available, the function:

1. Sets the working quantity/output word to zero.
2. Calls F-23e6231c5bdd with the record reference and stores its 32-bit result in a metadata word.
3. Sets the working reference word to the address of that metadata word.
4. Sets a second metadata word to zero.
5. Calls F-53082517a864 with:
   - The current processing state.
   - The current position.
   - The remaining quantity.
   - The address of the working quantity/output word.

The function does not explicitly pass the metadata address as a separate argument to this dependency. The later return value can expose that address.

The working quantity/output word is treated as an output from F-53082517a864. It is initialized to zero before the metadata dependency runs and before the processing dependency is called.

### 6. Interpret the processing result

#### Zero result

When F-53082517a864 returns zero:

1. The current state is cleared by calling the indirect operation at dispatch-table byte offset 8 with zero.
2. F-4bda01947d15 is called with the original context reference and the processing state used for the attempt.
3. The remaining quantity is replaced by the low 16 bits of:
   
   remaining quantity minus reported quantity

4. The current position is increased by the reported quantity.
5. Another attempt occurs if the new remaining quantity is nonzero.

The state-clear operation precedes the handoff call, and the handoff call precedes both arithmetic updates.

The reported quantity is read from working storage for these updates after F-4bda01947d15 returns. The supplied evidence does not establish that an unknown dependency cannot indirectly affect that storage after its address has escaped.

#### Distinguished nonzero result

When F-53082517a864 returns the numeric value 37, the function compares the reported quantity with the remaining quantity for that attempt.

- If they are equal, the loop terminates normally.
- If they differ, F-a817d8d58c10 is invoked on a nonreturning failure path.

On the accepted path, the function does not visibly clear the installed state, invoke F-4bda01947d15, advance the position, or reduce the remaining quantity.

#### Other nonzero result

Any nonzero result other than 37 invokes F-a817d8d58c10 on a nonreturning failure path.

The result is not visibly dereferenced. The distinguished value is therefore specified as a numeric discriminator, not as an object reference.

### 7. Perform final completion or return working words

After a skipped loop or normal loop termination, the function rereads the record’s 16-bit flag field.

If bit 0, mask 0x0001, is set, it calls the record’s indirect operation at dispatch-table byte offset 4, passing the record reference, and returns that operation’s 64-bit result.

This is a fresh flag read, not reuse of the earlier flag value. Changes made by intervening dependencies can therefore affect this decision.

If bit 0 is clear, the function returns the combined working words:

- With no processing attempt, the lower half contains the context reference and the upper half contains the original record reference.
- After a processing attempt, the lower half contains the address of the local metadata word, and the upper half contains the working quantity/output value as it exists at return.

The second case exposes a reference to function-local storage in the raw result. The supplied evidence does not establish that this reference is valid for dereference after return or that callers are intended to treat it as a pointer.

## Side effects

### Directly evidenced effects

- Reads a context field and record-linked flag storage.
- Invokes record-related descriptor dependencies in a fixed initial order.
- May query and update state through indirect operations.
- May create or obtain a replacement processing state.
- Passes writable local storage to F-53082517a864.
- Clears installed state before handing the previous state to F-4bda01947d15 on each zero processing result.
- May invoke a final record operation that determines the returned value.
- May terminate through either of two nonreturning paths.

The visible body contains no direct heap allocation, deallocation, or global assignment.

### Ownership and lifetime

Ownership is **documented_only** beyond the observable handoffs:

- A state obtained from F-b85dfaa750f8 is installed into the resolved object.
- A state associated with a zero processing result is first detached, then passed to F-4bda01947d15.
- Nothing supplied establishes whether that handoff releases, queues, transfers, resets, or retains the state.
- The final record operation must not be assumed to destroy or release the record.
- No dependency may be assumed to preserve global state, pointed-to data, or previously exposed storage unless a separate contract establishes it.

### Arithmetic and progress boundaries

The remaining-quantity update explicitly masks to 16 bits. It therefore wraps modulo 65,536 rather than rejecting underflow or overflow.

The current-position update uses the visible 32-bit arithmetic representation, with no explicit range check. Carry beyond that width is not retained in the represented result.

There is no visible requirement that reported progress be positive, nonzero, or bounded by the remaining quantity. In particular:

- Zero progress after a zero processing result can leave both loop-control quantities unchanged.
- Excess progress can wrap the remaining quantity instead of terminating.
- The initial quantity is not visibly narrowed to 16 bits until the first zero-result update.
- Normal loop termination is not guaranteed.

## Dependencies

| Dependency | Observable role |
|---|---|
| F-b1020422996f | Obtains the initial remaining quantity from the record. |
| F-2dd0aac96475 | Obtains the initial current position from the record. |
| F-407933f268e5 | Obtains the lookup key from the record. |
| F-b37968c9f644 | Resolves a state object using the context-derived lookup reference and key. |
| F-09ba55c206ac | Called when the fresh-state flag is set but a nonzero state is already present. |
| F-feee9156a257 | Nonreturning continuation of that fresh-state failure path. |
| F-b85dfaa750f8 | Supplies a processing state when none is installed. |
| F-23e6231c5bdd | Supplies per-attempt metadata stored locally. |
| F-53082517a864 | Processes the current position and quantity; returns a discriminator and receives writable quantity storage. |
| F-4bda01947d15 | Receives the context and detached state following a zero processing result. |
| F-a817d8d58c10 | Nonreturning failure for unavailable state or an unacceptable processing result. |

The state-query, state-installation, and final record operations are indirect dependencies without supplied opaque function IDs. Their identities and internal contracts remain unresolved.

## Acceptance examples

These examples are conditional static acceptance criteria, not executed tests.

1. **No initial quantity.**  
   Given an initial quantity of zero and valid structures for all accesses, no processing attempt occurs. Initial descriptor and lookup calls still occur. Bit 1 can still trigger state operations. If the final bit 0 is clear, the returned upper word is the record reference and the lower word is the context reference.

2. **Accepted distinguished result.**  
   Given remaining quantity 12, a valid current state, and processing result 37 with reported quantity 12, the loop stops normally. No zero-result state handoff occurs for that attempt.

3. **Rejected distinguished result.**  
   Under the same conditions, result 37 with reported quantity 11 enters F-a817d8d58c10. The final completion operation is not reached under the nonreturning model.

4. **Rejected other result.**  
   A processing result of 1 enters F-a817d8d58c10 regardless of the reported quantity.

5. **Partial zero-result progress.**  
   Given quantity 10, position 4096, result zero, and a reported quantity that remains 4 through the handoff, the state is cleared and handed off before the next attempt. The next quantity is 6 and the next position is 4100.

6. **Exhaustion through a zero result.**  
   Given quantity 10 and stable reported quantity 10 after a zero result, the state is cleared and handed off, the remaining quantity becomes zero, and no further attempt occurs.

7. **Wrapped subtraction.**  
   Given quantity 1 and stable reported quantity 2 after a zero result, the next quantity is 65,535, not an error and not zero.

8. **Fresh-state violation.**  
   If bit 1 is set and the initial state query returns nonzero, F-09ba55c206ac precedes F-feee9156a257. Processing is not reached under the nonreturning model.

9. **Changed completion flag.**  
   If an intervening dependency changes bit 0 before the final flag read, the final decision follows the reread value, not the entry value.

## Uncertainties

- **documented_only:** Semantic meanings and units of the initial quantity, position, lookup key, metadata, and state values.
- **documented_only:** Whether the represented 64-bit return is a deliberate public result format or reflects an incompletely recovered calling convention.
- **documented_only:** Intended use of the returned address into local storage.
- **documented_only:** Ownership, retention, release behavior, and global effects of all dependencies.
- **documented_only:** Whether indirect operations validate their receivers or can fail independently.
- **documented_only:** Any stronger progress invariant imposed by F-53082517a864.
- **documented_only:** Whether callers constrain the initial quantity to 16 bits or prohibit arithmetic wraparound.
- **documented_only:** Effects if dependencies retain the exposed output-storage address.
- **documented_only:** Behavior outside the readable-memory and callable-dispatch preconditions.

An inferred intent is incremental input processing with state rollover. That interpretation does not replace the explicit result discrimination, masked arithmetic, or state-operation ordering established above.

## Evidence

The supplied static body establishes call order, argument construction, flag masks, dispatch offsets, branching conditions, explicit arithmetic, local-word assignments, and represented result construction.

The evidence carries warnings about unstable type propagation and incomplete recovery of the final indirect transfer. Accordingly, pointer-like quantity types, the return convention, and the apparent local-storage address result require particular caution. No stronger application-level meaning is asserted for them.

The two nonreturning failure behaviors follow the supplied static annotations; they have not been independently observed. No runtime execution, dependency implementation inspection, or acceptance testing has been performed.
