<!-- Trusted provenance: F-eaa0656aab7c; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-eaa0656aab7c

## Purpose

F-eaa0656aab7c coordinates conditional checks against two regions of a caller-supplied context, then either returns directly, invokes a secondary operation, or dispatches an indirect callback.

The established behavior is a control-flow contract, not a semantic classification of the checked values:

- A matching discriminator enables an initial check against the context region at byte offset 112.
- A nonzero result from that initial check causes F-eddad957e718 to be called with two zero arguments, bypassing all remaining checks and the indirect callback.
- Otherwise, a check against the context region at byte offset 84 determines whether another check is needed.
- The final check, when reached, uses the context region at byte offset 112.
- The indirect callback occurs if the offset-84 check returns nonzero, or if the final offset-112 check returns nonzero.

On every normally returning path, the function returns a 64-bit value whose low 32 bits are 1. Its high 32 bits are either the original record address, a subsequently loaded record field, or a value left in a writable local slot by F-271eefe0efc5.

The domain meaning of the checks, the secondary operation, and the callback is unknown. Interpreting them as lookup, removal, registration, notification, or error handling would exceed the available evidence.

## Interface

### Inputs

There are four inputs:

| Input | Established representation | Use |
|---|---|---|
| Context | Pointer to a structure accessed through 32-bit words | Supplies two interior addresses, an indirect-dispatch base, and a callback argument |
| Record address | 32-bit address-bearing value | Identifies memory read at byte offsets 8 and 12; initially also supplies the high return word |
| Third input | Opaque 32-bit value | Copied into a local slot only on the discriminator-matching path |
| Fourth input | Opaque 32-bit value | Initially copied into a local slot; overwritten before that slot is passed to a dependency |

No explicit length, capacity, ownership flag, or validity indicator accompanies either address.

### Required memory layout

The visible context accesses are:

| Byte offset from context | Access |
|---|---|
| 0 | Reads a 32-bit dispatch-table address on the callback path |
| 44 | Reads a 32-bit value supplied as the callback’s third argument |
| 84 | Forms an interior address passed to F-271eefe0efc5 |
| 112 | Forms an interior address passed to F-271eefe0efc5 |

The visible record accesses are:

| Byte offset from record address | Access |
|---|---|
| 8 | Reads a 32-bit discriminator |
| 12 | Reads a 32-bit payload word, potentially more than once |

For indirect dispatch, a callable address is obtained from byte offset 88 within the dispatch table identified by the context’s first word.

These offsets do not establish the full size or type of either structure. In particular, the storage requirements of the context regions at offsets 84 and 112 depend on F-271eefe0efc5.

### External values

Two external 32-bit values participate:

- A discriminator reference, compared against the record word at offset 8.
- A callback argument, read for the indirect callback.

Their values and meanings are not supplied. The contract does not treat either as immutable across dependency calls.

### Output

The result is 64 bits wide:

- Bits 0–31 are exactly the bit pattern for 1.
- Bits 32–63 are the final contents of a particular 32-bit local return slot.

That return slot initially contains the original record address. It is replaced by a fresh read of the record word at offset 12 only if the offset-84 check returns zero. Its address is then passed to the final check, which may modify it.

This is a bitwise composition, not a signed widening of the high word. For example, a high word of `0x80000000` produces `0x8000000000000001`.

### Preconditions

For defined normal operation:

- Each actually executed record read must address readable storage.
- The context must support the accesses and interior-address formation on the executed path.
- Storage and alignment must meet the requirements of the target environment and dependencies.
- If indirect dispatch is reached, the dispatch-table address and its entry at offset 88 must identify valid readable storage and a compatible callable target.
- Memory required after a dependency call must remain valid at the time of the later access.

There are no visible null checks, bounds checks, or validity checks. Address arithmetic is not guarded against overflow. This contract does not prescribe a portable result for invalid addresses, unsuitable alignment, or address overflow.

## Behavior

### Initial state and discriminator decision

The function first initializes the 32-bit return slot with the original record address. A separate local slot receives the fourth input.

It then reads the record discriminator at byte offset 8 and compares it with the external discriminator reference.

The discriminator is checked only once. No later branch revalidates it, even if a dependency changes the record or the external reference.

### Optional initial check

If the discriminator matches:

1. The function reads the record word at byte offset 12 into a local slot.
2. It copies the third input into another local slot.
3. It calls F-271eefe0efc5 with:
   - the context interior address at byte offset 112; and
   - the address of the local slot containing the freshly read record word.

The dependency’s result is tested only for zero versus nonzero. No distinction is made between positive and negative nonzero values.

If this result is nonzero, the function calls F-eddad957e718 with the context and two zero-valued arguments. It then returns without performing the offset-84 check, the final offset-112 check, or the indirect callback.

On this path, the function itself has not replaced the initial high return word. Consequently, under ordinary access limited to the explicitly supplied local slot, the high word remains the original record address. The internals of both dependencies remain unknown.

If the initial check returns zero, processing continues to the offset-84 check.

If the discriminator does not match, the initial check and F-eddad957e718 are both skipped.

### Offset-84 check

On every path not terminated by the special nonzero result, the function performs a new read of the record word at byte offset 12. It places this value in a local slot and calls F-271eefe0efc5 with:

- the context interior address at byte offset 84; and
- the address of that local slot.

This read is not specified to reuse the value from the optional initial check. If that earlier dependency changed the record payload, this check receives the later value.

If the offset-84 check returns nonzero, the function proceeds directly to the indirect callback. It does not perform the final offset-112 check, and it has not explicitly replaced the original high return word.

If the offset-84 check returns zero, processing continues to the final check.

### Final offset-112 check

After a zero result from the offset-84 check, the function again reads the record word at byte offset 12.

This time the value replaces the return slot itself. The function then calls F-271eefe0efc5 with:

- the context interior address at byte offset 112; and
- the address of the return slot.

Because the return slot is passed by address, a write by the dependency to that slot directly affects the high 32 bits of the eventual return value. The function does not restore the loaded record word afterward.

A zero result from this check causes a direct return without the callback. A nonzero result causes indirect dispatch.

The result controlling dispatch and the value retained in the writable slot are separate observations. A zero result does not establish that the slot was unchanged.

### Indirect callback

The indirect callback is reached under exactly these visible conditions:

- The offset-84 check returned nonzero; or
- The offset-84 check returned zero and the final offset-112 check returned nonzero.

The callback receives three arguments:

1. The context pointer.
2. The external callback argument.
3. The 32-bit context word at byte offset 44.

The callable address is resolved using the context’s first word as a dispatch-table address and selecting the entry at byte offset 88.

These callback-related values are obtained after the preceding checks. They are not visibly cached before those calls. Changes made by a preceding dependency can therefore affect dispatch or callback arguments.

The evidence does not establish the relative evaluation order among the dispatch-target read and the separate callback-argument reads. It establishes that the required reads precede invocation.

The callback’s return value does not contribute to this function’s returned bit pattern.

### Normal completion

After any applicable operation or callback returns normally, the function composes the result from the current return slot and the constant low word 1.

There is no visible alternate normal return encoding for a failed check. Check results choose control-flow paths; they do not replace the low return word.

## Side effects

The function’s explicit data writes are to its own local slots. It contains no direct assignment to the context, record, or external values.

It can nevertheless cause externally visible effects through:

- Up to three calls to F-271eefe0efc5.
- At most one call to F-eddad957e718.
- At most one indirect callback.

The secondary operation and indirect callback are mutually exclusive within one invocation.

The maximum of three check calls occurs when the discriminator matches, the initial check returns zero, and the offset-84 check also returns zero. Without a discriminator match, there are at most two check calls.

Pointers passed to dependencies include context interior addresses and addresses of temporary local storage. The function performs no visible allocation, deallocation, reference-count adjustment, or ownership transfer. Whether a dependency allocates, releases, mutates, or retains anything is unknown.

The local-slot addresses do not represent storage guaranteed to outlive this invocation. Persistent retention of those addresses is not supported by the visible interface.

No global-preservation guarantee is established for any callee. Likewise, no guarantee is established that a callee preserves the record, context, dispatch table, or unrelated memory. The described read sequence is significant precisely because later reads may observe changes made by earlier calls.

## Dependencies

### F-271eefe0efc5

Used as a zero/nonzero decision operation.

Each invocation receives a context interior address and a writable pointer to a 32-bit local slot. The visible call sites use context offsets 84 or 112.

Its result is observed as a 32-bit zero/nonzero value. Its semantics, memory access extent, mutation behavior, ownership behavior, and failure modes are unknown.

The source also initializes other local slots near the call sites. That does not establish a multiword argument layout. In particular, the third input must not be specified as a second field consumed by this dependency without additional layout or callee evidence.

### F-eddad957e718

Called only after a discriminator match followed by a nonzero result from the initial offset-112 check.

It receives the context and two zeros. Its return value is ignored. Its semantic purpose and side effects are unknown.

### Indirect dependency

The indirect target has no supplied opaque function ID. It is selected through the context’s dispatch table at byte offset 88.

Its three visible arguments are described above. Its return value is ignored. Its behavior, calling requirements beyond those visible arguments, and memory effects are unknown.

## Acceptance examples

These examples are conditional static acceptance criteria, not executed tests. Unless otherwise stated, dependencies return normally, modify only the explicitly identified writable slot where specified, and leave other required storage valid.

1. **Matching discriminator with an immediately nonzero check.**  
   The initial offset-112 check returns a nonzero value. F-eddad957e718 must receive the context and two zeros. Neither later check nor the callback occurs. The return value has the original record-address bits as its high word and 1 as its low word.

2. **Nonmatching discriminator with a nonzero offset-84 result.**  
   The initial check is skipped. The offset-84 check runs and returns nonzero. The callback runs once; the final offset-112 check does not run. The high return word remains the original record address.

3. **Nonmatching discriminator with two zero results.**  
   The offset-84 check returns zero. The record payload is read again into the return slot, and the final offset-112 check returns zero without modifying that slot. No callback occurs. If the latest payload is `0x12345678`, the return value is `0x1234567800000001`.

4. **Final check changes the high return word.**  
   The offset-84 check returns zero. The final check writes `0xDEADBEEF` into its supplied slot and returns zero. No callback occurs, and the returned value is `0xDEADBEEF00000001`.

5. **Final check returns nonzero after changing the slot.**  
   The final check writes a new 32-bit value and returns any nonzero result, including a negative result under a signed interpretation. The callback runs once. The dependency-written value remains the high return word under the stated preservation conditions.

6. **Payload mutation between checks.**  
   The discriminator matches. The initial check changes the record payload from A to B and returns zero. The offset-84 check receives B. If it changes the payload to C and returns zero, the final check receives a slot initialized to C. The function must not substitute the first payload read for either later read.

7. **Dispatch state changes during a check.**  
   A check changes the context’s dispatch-table address or word at offset 44 before returning a result that leads to the callback. The callback uses the subsequently read state, not a value cached before the check. This criterion does not impose an unsupported ordering among the callback-preparation reads themselves.

8. **Changes to the third and fourth inputs.**  
   With all observable memory and dependency behavior otherwise fixed, the visible scalar operations do not use these inputs to choose branches or form the return value. A stronger claim of complete behavioral independence is not established: the access extent and local-layout assumptions of F-271eefe0efc5 are unknown.

## Uncertainties

The following remain **documented_only**:

- Semantic meanings of the discriminator, payload, context regions, and external callback argument.
- The full layouts and required extents of the context, record, and dispatch table.
- Whether F-271eefe0efc5 accesses more than the addressed 32-bit slot or relies on surrounding local storage.
- Whether the third input has an indirect role through such surrounding storage.
- All dependency ownership, allocation, release, retention, and global-mutation behavior.
- Callback identity and semantic purpose.
- Fault, exception, nonreturning, concurrency, and reentrancy behavior.
- Platform guarantees for invalid addresses, alignment, or address-arithmetic overflow.

A lookup-and-action interpretation is plausible but remains inferred intent, not part of the established contract.

## Evidence

Established static evidence comprises the supplied function body and dependency-ID mapping. It supports the word widths, byte offsets, conditional call sequence, repeated payload reads, writable final return slot, indirect dispatch, and 64-bit return composition described here.

The pointer-bearing accesses support a 32-bit address model for the visible layout; they do not independently establish a complete platform ABI.

No dependency bodies, runtime observations, or runtime tests were supplied. No runtime tests have been performed. Acceptance examples specify conditional expected behavior derived from the visible operations; they are not claims of observed execution.
