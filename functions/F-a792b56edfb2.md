<!-- Trusted provenance: F-a792b56edfb2; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-a792b56edfb2

## Purpose

F-a792b56edfb2 conditionally extracts a 16-bit selector from an indirectly referenced record, passes that selector to one dependency, performs up to two conditional calls to another dependency, and then uses the saved selector to select a 32-bit argument for an indirect invocation.

Two ordered gates control whether processing proceeds. A zero result from either gate causes an immediate return. Once both gates succeed, selector extraction is unconditional: the function does not validate the pointer chain before reading it.

The established behavior is this gated sequence of reads and calls. Describing the operation as resource activation, cleanup, notification, or event dispatch would assign semantics not established by the supplied evidence.

## Interface

**Input:** One machine-sized value used as the address of a caller-supplied object. The recovered interface represents this value as a 32-bit integer rather than a typed object pointer.

**Output:** No explicit return value. The function either returns early, completes the downstream call sequence and returns, or inherits any abnormal behavior of its memory accesses or dependencies.

**Relevant object layout:**

| Offset from object address | Access width | Established use |
|---|---:|---|
| `0x128` | 32 bits | Argument supplied to F-18bc1d07eb44 after the first gate succeeds. |
| `0x240` | 32 bits | Tested against zero after F-6443787f9616 returns. |
| `0x244` | 32 bits | Initially interpreted as a pointer for selector extraction; later reread and tested against zero. |

The selector is reached through two pointer loads:

1. Read the pointer-sized value stored at object offset `0x244`.
2. Read another pointer-sized value at the address obtained from the first load.
3. Read an unsigned 16-bit value at offset `0x78` from the address obtained from the second load.

The evidence uses 32-bit pointer and word representations. The 16-bit selector is zero-extended to 32 bits, giving a range of 0 through 65,535.

Two external memory locations also participate in the final operation:

- One supplies a table-base value. Entries are accessed at a four-byte stride and read as 32-bit values.
- One supplies the indirect invocation target.

Their addresses are not part of the public contract. Their initialization, lifetime, mutability, and meanings are not established.

**Preconditions are path-dependent.** A call rejected by the first gate does not require readable object fields on the shown path. A call rejected by the second gate requires the read at offset `0x128`, but not the selector pointer chain. A call passing both gates requires that chain to be readable when it is accessed. Successful continuation additionally requires readable condition fields, an accessible selected table entry, and a valid indirect invocation target at the times those operations occur.

No ownership transfer, allocation, or release is explicitly represented by this interface.

## Behavior

### 1. Evaluate the first gate

The function calls F-4a92cc2e451c with no explicit arguments.

If its 32-bit result is zero, F-a792b56edfb2 returns immediately. In the supplied body, no object-field read, selector extraction, or later dependency call precedes this return.

Any nonzero result permits continuation. This includes values that would be negative under a signed 32-bit interpretation; the decision is equality to zero, not positivity.

This gate is called before the function explicitly reads the input object. Its internal reads, writes, and other effects remain unknown.

### 2. Evaluate the second gate

After the first gate returns nonzero, the function reads the 32-bit value at object offset `0x128` and passes that value to F-18bc1d07eb44.

If F-18bc1d07eb44 returns zero, the function returns immediately. No selector extraction or later call is shown on this path.

If it returns any nonzero value, processing continues.

The offset-`0x128` read occurs after F-4a92cc2e451c returns. A contract must therefore not substitute an object-field value captured before that call: the dependency is not established to preserve the object or any other memory.

### 3. Extract and retain the selector

The function follows the pointer chain rooted at object offset `0x244` and reads the unsigned 16-bit selector at the final record’s offset `0x78`.

There is no preceding zero check on either pointer in this chain. In particular, the later nonzero test of object offset `0x244` does not protect this earlier dereference.

The selector is zero-extended to 32 bits and retained for both:

- The argument to F-6443787f9616.
- The index used for the final table lookup.

A selector with bit 15 set is not sign-extended. For example, the 16-bit value `0xFFFF` becomes 65,535, not −1.

The function then calls F-6443787f9616 with that retained selector. No return value from this dependency is used by the visible body.

Later changes to the selector’s original memory location do not cause a second selector read. Subject to the recovered representation, both uses depend on the same extracted value.

### 4. Perform the first conditional call

After F-6443787f9616 returns, the function reads the 32-bit field at object offset `0x240`.

If that field is nonzero, it calls F-66819ba2201a with no explicit arguments. If the field is zero, it skips this call.

The condition uses the field’s value after F-6443787f9616 has run. It is not established to use the object’s entry-state value.

### 5. Perform the second conditional call

After the first conditional call has either completed or been skipped, the function rereads the 32-bit field at object offset `0x244` and tests it against zero.

If this newly read value is nonzero, it calls F-66819ba2201a with no explicit arguments. Otherwise, it skips this call.

The two conditions are independent and ordered. There can be zero, one, or two calls to F-66819ba2201a. The second condition is not evaluated in advance of the first call.

Consequently, if the first invocation of F-66819ba2201a changes offset `0x244`, the second condition observes the resulting value. The same applies to changes made earlier by F-6443787f9616.

The second condition only tests the field against zero. It does not follow the pointer chain again.

### 6. Select an argument and invoke the indirect target

After both conditional-call opportunities, the function uses the retained selector to access an externally based table:

- Entry stride: four bytes.
- Entry width: 32 bits.
- Selected byte offset: four times the selector.
- Selected entry: passed as the sole explicit argument to the indirect target.

For selectors from 0 through 65,535, the computed byte offset ranges from 0 through 262,140, or `0x3FFFC`. Multiplication by four therefore does not overflow an unsigned 32-bit value. This fact does not establish that adding the offset to the table base yields a valid address or that the table contains that entry.

There is no visible bounds check, selector remapping, or special treatment for selector zero or 65,535.

The table-base value and indirect target are used after the preceding calls. The evidence does not establish that either remains unchanged across those calls. An implementation must not assume that entry-state global values are the values used here.

The relative evaluation order of the final target load and the table-related loads is not established by the supplied representation. The established ordering is that the final operation follows the earlier conditional-call sequence.

No explicit result from the indirect invocation is consumed. The recovered body then returns, although the exact machine-level nature of this final transfer remains uncertain.

## Side effects

The visible body contains no explicit store to the input object, selector record, table, or global state.

Its direct observable actions are memory reads and invocations of dependencies. Those invocations may produce additional effects. The supplied evidence does not establish that they:

- Preserve global state or the input object.
- Leave the selector pointer chain or table intact.
- Allocate or release memory.
- Return normally.
- Avoid callbacks, reentrancy, or external I/O.

The function therefore has no established pure, read-only, or resource-neutral guarantee despite having no explicit stores.

Ownership remains unspecified. Passing the offset-`0x128` value, selector, or selected table entry does not by itself prove borrowing or transfer. Similarly, the two no-argument conditional calls cannot be described as releasing the fields that triggered them: neither field is passed explicitly.

No error code, diagnostic, retry, rollback, or recovery path is shown. Zero gate results cause ordinary early returns. Invalid memory, invalid transfer targets, and dependency failures have no locally specified handling.

## Dependencies

| Opaque ID | Explicit inputs | Visible role |
|---|---|---|
| F-4a92cc2e451c | None | First gate; zero stops all subsequent processing. |
| F-18bc1d07eb44 | One 32-bit value from object offset `0x128` | Second gate; zero stops selector processing and later calls. |
| F-6443787f9616 | Zero-extended 16-bit selector, supplied as 32 bits | Called once after successful selector extraction. |
| F-66819ba2201a | None | Called zero, one, or two times according to two ordered field tests. |

The final dependency is an indirect target loaded from external state; no opaque function ID is supplied for it. Its sole visible argument is the selected 32-bit table entry. Its calling convention, accepted argument semantics, side effects, ownership rules, and return behavior are documented_only unknowns.

The direct dependencies’ roles above describe only their use by F-a792b56edfb2, not their internal contracts.

## Acceptance examples

These examples are conditional static acceptance criteria, not executed tests. Unless stated otherwise, dependencies return normally, required memory is valid, and no unmentioned mutation occurs.

1. **First gate rejects.**  
   F-4a92cc2e451c returns zero. The function returns without explicitly reading the input object or calling any later dependency.

2. **Second gate rejects.**  
   The first gate returns nonzero. The function reads object offset `0x128` and supplies that value to F-18bc1d07eb44. If the latter returns zero, there is no selector-chain access and no call to F-6443787f9616, F-66819ba2201a, or the final indirect target.

3. **No conditional calls.**  
   Both gates succeed and selector extraction yields 7. F-6443787f9616 receives 7. If the subsequent reads of offsets `0x240` and `0x244` each yield zero, neither conditional call occurs. The final selected argument is the 32-bit entry at table byte offset 28. For this scenario to be valid, offset `0x244` must have supported the earlier pointer-chain read and then become zero before its later condition is evaluated.

4. **Both conditions hold.**  
   Both gates succeed. The offset-`0x240` condition reads nonzero, causing one call to F-66819ba2201a. After that call returns, the offset-`0x244` condition also reads nonzero, causing a second call. The final indirect invocation follows both calls.

5. **The first conditional call suppresses the second.**  
   The first condition reads nonzero. Its invocation of F-66819ba2201a changes object offset `0x244` to zero. The function then reads zero for the second condition and does not make a second invocation.

6. **Unsigned upper boundary.**  
   Selector extraction yields `0xFFFF`. F-6443787f9616 receives 65,535. The final lookup uses byte offset `0x3FFFC`. Successful completion requires a readable entry there; no visible check establishes it.

7. **Selector storage changes after extraction.**  
   Selector extraction yields 3. F-6443787f9616 changes the underlying selector storage to 9. The final lookup still uses the retained value 3, selecting byte offset 12.

8. **Null selector-chain root after successful gates.**  
   Both gates succeed, but object offset `0x244` is zero when selector extraction begins. No normal-return guarantee applies. The function attempts the pointer-chain access before its later zero test; acceptance must not require a graceful skip.

## Uncertainties

- **documented_only — Domain meaning:** The object fields, selector, table entries, and final operation have no established application-level meaning.
- **documented_only — Dependency effects:** Internal state changes, memory ownership, failure modes, and preservation guarantees are unknown.
- **documented_only — Final control transfer:** Recovery metadata reports uncertainty around the final indirect transfer. The representation supports a selected argument and an indirect invocation, but not a definitive claim about ordinary call versus tail-transfer mechanics.
- **documented_only — Memory rules:** Object sizes, table bounds, alignment requirements, byte order, address-wrap behavior, and invalid-access outcomes are not specified.
- **documented_only — Execution environment:** Initialization, synchronization, concurrent mutation, and reentrancy rules are not supplied.
- **Inferred intent:** A selector-indexed downstream operation is plausible. Stronger interpretations, including cleanup or resource release, are unsupported.

## Evidence

The supplied static representation establishes the two zero-result gates, field offsets and access widths, pointer-chain shape, unsigned selector extension, ordered conditional calls, four-byte table indexing, and final indirect invocation representation.

Recovery metadata limits confidence in the exact mechanics of the final transfer. No dependency implementations or runtime observations were supplied.

No runtime tests have been performed. The acceptance examples specify behavior conditional on the stated memory and dependency assumptions; they are not empirical results.
