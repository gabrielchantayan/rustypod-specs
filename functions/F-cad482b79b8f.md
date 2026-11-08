<!-- Trusted provenance: F-cad482b79b8f; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-cad482b79b8f

## Purpose

F-cad482b79b8f synchronously offers an optional input object to a prioritized set of recipients and returns the first nonzero result, or zero if no invoked recipient supplies one.

Recipient selection has three stages:

1. Recipients obtained from an optional context-provided collection.
2. An explicitly supplied recipient, when present.
3. Otherwise, a context-provided traversal of default recipients.

A nonzero result from the collection stage suppresses both later stages. An explicit recipient suppresses the default traversal even when that recipient returns zero.

On every visible normal-return path, a nonnull input object receives one final virtual operation before the saved result is returned. The meaning of that operation—release, destruction, notification, or something else—is not established.

This is a static behavioral contract. No runtime tests have been performed.

## Interface

### Inputs

Two pointer-sized arguments are visible:

- **Input object:** nullable; forwarded to recipient operations and, conditionally, inspected through its own virtual interface.
- **Explicit recipient:** nullable; used only if the collection stage produces no nonzero result.

The represented interface uses 32-bit addresses and 32-bit integer results. There is no established signedness-based interpretation of recipient results: zero means continue or unhandled, and any nonzero 32-bit value terminates the relevant dispatch path and is returned unchanged.

Nullability is supported only where explicitly described. Other pointers reached through contexts, objects, tables, and helper outputs must be valid for the accesses performed.

### Output

The function returns a 32-bit result:

- The first nonzero result from a collection recipient, if any.
- Otherwise, the explicit recipient’s result, if an explicit recipient was supplied.
- Otherwise, the first nonzero result from an eligible default recipient.
- Otherwise, zero.

The final input-object operation does not replace this saved result.

### Observable object layouts

The following byte offsets are established by the visible accesses. They describe only the fields required by this function, not complete object definitions.

| Object | Offset | Required interpretation |
|---|---:|---|
| Context returned by F-3fdd14ecd250 | 0x2c | 32-bit collection-associated value; zero skips collection setup |
| Context returned by F-3fdd14ecd250 | 0x28 | 32-bit initial default-recipient address |
| Input object | 0x00 | Address of virtual dispatch table |
| Input dispatch table | 0x04 | Final input-operation entry |
| Input dispatch table | 0x14 | Input classification-operation entry |
| Collection recipient | 0x00 | Address of virtual dispatch table |
| Collection-recipient dispatch table | 0x34 | Collection handling entry |
| Explicit or default recipient | 0x00 | Address of virtual dispatch table |
| Recipient dispatch table | 0x30 | Recipient handling entry |
| Default-recipient dispatch table | 0x1c | Successor-producing entry |
| Default recipient | 0x0c | 32-bit associated-record address |
| Associated record | 0x0c | 32-bit filtering field |

The classification and successor operations expose two-word, 64-bit results in the supplied representation. For this function’s decisions:

- Only the classification result’s low 32-bit word determines whether filtering is enabled.
- Only the successor result’s low 32-bit word determines the next recipient address.
- The upper words do not determine eligibility, termination, or the returned result.

No application-level meaning for those upper words is established.

## Behavior

### 1. Collection stage

The accumulated result begins at zero.

F-3fdd14ecd250 is called to obtain a context address. The function reads the 32-bit field at context offset 0x2c.

If that field is zero, collection setup, iteration, and collection cleanup are skipped.

If it is nonzero:

1. The same context field is read again for the setup argument. The branch-check read and setup-argument read are distinct accesses.
2. F-538044b67754 is called with:
   - The address of a local 20-byte working area.
   - The collection-associated value from the second read.
   - The exact 32-bit bit pattern 0xfffffffe.
3. F-6e4651b532dc is repeatedly called with the working-area address and the address of a local recipient-pointer output slot.
4. A zero result from F-6e4651b532dc ends collection enumeration without invoking a recipient for that step.
5. A nonzero result causes the produced recipient to receive its virtual operation at table offset 0x34. The recipient itself and the input-object argument are passed to that operation.
6. A zero recipient result resumes enumeration. A nonzero recipient result becomes the saved result and ends collection enumeration.
7. F-944b595e25e9 is called once on the working area after either visible enumeration exit.

After collection cleanup:

- A nonzero saved result proceeds directly to the final input operation.
- A zero saved result proceeds to fallback selection.

The visible function does not validate a successful enumeration output before dereferencing it. Every nonzero enumeration result therefore requires a valid recipient pointer and a valid dispatch entry at offset 0x34.

The setup constant can be interpreted as −2 in signed 32-bit arithmetic, but its semantic meaning is unknown. It must not be replaced by an inferred enumeration policy.

### 2. Explicit-recipient fallback

If the collection stage produced no nonzero result and the explicit-recipient argument is nonnull, that recipient receives its virtual operation at table offset 0x30.

The arguments are the explicit recipient itself and the original input-object pointer. The operation’s complete 32-bit result becomes the saved result.

Default traversal is not attempted afterward, including when this result is zero.

This path does not call the input classification operation and does not obtain the default-recipient head.

### 3. Default traversal setup

Default traversal occurs only when:

- The collection stage produced no nonzero result; and
- No explicit recipient was supplied.

F-3fdd14ecd250 is called again. The function reads the 32-bit default-recipient address at offset 0x28 of the newly returned context.

This is not a reuse of the first context lookup. The two calls may return different addresses or expose different state. No preservation guarantee for intervening dependencies is established.

The default-recipient address is captured **before** input classification.

Filtering is then selected:

- For a null input object, filtering is disabled and no classification operation is invoked.
- For a nonnull input object, its virtual operation at table offset 0x14 is invoked once. A zero low 32-bit return word disables filtering; a nonzero low word enables filtering.

A classification operation that changes the context’s default-head field does not, by that change alone, replace the already captured starting address. Mutations to recipient objects or other state can still affect later reads.

### 4. Default-recipient traversal

Traversal starts at the captured default-recipient address.

A null current recipient ends traversal. If no nonzero result has been obtained, the result remains zero.

For each nonnull current recipient, eligibility is determined as follows.

#### Filtering disabled

The recipient is eligible without inspecting its associated-record filtering field and without calling F-35b8bd5fa9e0.

#### Filtering enabled

The function:

1. Reads the associated-record address at recipient offset 0x0c.
2. Reads the 32-bit field at offset 0x0c within that associated record.
3. Treats a zero field as immediately eligible.
4. For a nonzero field, calls F-35b8bd5fa9e0 with the input object.
5. Treats a nonzero dependency result as eligible and a zero result as ineligible.

This dependency is evaluated separately for each encountered recipient whose filtering field is nonzero. Its result is not cached across recipients.

**The associated-record dereference precedes any zero-address termination check on that record.** With filtering enabled, a null associated-record address is therefore not a supported safe terminator.

#### Handling and advancement

An eligible recipient receives its virtual operation at table offset 0x30, with itself and the input object as arguments.

- A nonzero result is saved and immediately ends traversal.
- A zero result allows traversal to continue.

An ineligible recipient is not invoked; the saved result remains zero.

When traversal has not ended with a nonzero handling result, the function reads the current recipient’s associated-record address at offset 0x0c:

- If it is zero, traversal ends with zero.
- If it is nonzero, the current recipient receives its virtual successor operation at table offset 0x1c.
- The low 32-bit word of that result becomes the next recipient address.

The post-handling associated-record read is a fresh read. A handler or eligibility dependency may have changed that field before it is used to decide whether to advance.

The associated record is not itself followed as the next recipient. Its nonzero value permits the successor operation; that operation supplies the next address.

There is no visible cycle detection, traversal limit, or duplicate suppression.

### 5. Final input operation and return

After dispatch finishes, a nonnull input object receives its virtual operation at table offset 0x04.

This occurs after collection cleanup, explicit handling, or default traversal, as applicable. It occurs for both zero and nonzero saved results.

A null input object receives no final operation.

The function then returns the saved dispatch result. The return value, if any, of the final operation is ignored.

These guarantees concern normal control flow. If a dependency or virtual operation faults, fails to return, or performs a nonlocal transfer, later cleanup and finalization are not established.

## Side effects

### Reads and writes directly visible here

The function reads context fields, object dispatch pointers, dispatch entries, and the default traversal’s associated-record fields.

It maintains local state consisting of:

- A zero-initialized saved result.
- A 20-byte collection working area.
- A recipient-pointer output slot passed to enumeration.
- Current-recipient and filtering state.

No direct write to a context field, input-object field, recipient field, or associated-record field is visible. No direct heap allocation or deallocation is visible.

Dependencies and virtual operations may perform additional reads, writes, allocations, or releases. Their absence from this body does not establish their absence from execution.

### Ordering commitments

The normal-path ordering includes:

- Collection setup before enumeration.
- Each successful enumeration step before its corresponding collection-recipient operation.
- Collection cleanup before any fallback handling.
- Default-head capture before input classification.
- Per-recipient eligibility evaluation before that recipient’s handling operation.
- A zero handling result or skipped handler before the post-step associated-record check.
- That check before any successor operation.
- All dispatch processing before the final input operation.

### Ownership and lifetime

Ownership transfer cannot be established from the visible calls alone.

In particular:

- The final input operation must not be labeled a release or destructor without additional evidence.
- Collection cleanup must not be assumed to destroy or release enumerated recipients.
- No retain-like operation is visible.
- Recipient handling must not be assumed to preserve the input object, recipients, contexts, or global state.
- Classification and eligibility dependencies must not be assumed pure.

For a normal execution, all objects and dispatch entries needed by subsequent accesses must remain valid through those accesses. For example, if recipient handling invalidates the input object, the later final operation on that same nonnull pointer cannot be assumed safe.

## Dependencies

| Opaque ID or virtual entry | Visible role | Required behavior for the described path |
|---|---|---|
| F-3fdd14ecd250 | Obtain a context | Return an address readable at the field used by that call |
| F-538044b67754 | Initialize collection working state | Accept the 20-byte working area, collection value, and 0xfffffffe argument; normal return allows enumeration |
| F-6e4651b532dc | Advance collection enumeration | Zero ends enumeration; nonzero requires a valid recipient in the output slot |
| F-944b595e25e9 | Finish collection working state | Accept the initialized working area on either visible enumeration exit |
| F-35b8bd5fa9e0 | Conditional eligibility decision | Zero skips the current default handler; nonzero permits it |
| Collection entry at 0x34 | Offer input to an enumerated recipient | Return a 32-bit zero/nonzero dispatch result |
| Recipient entry at 0x30 | Offer input to an explicit or default recipient | Return a 32-bit zero/nonzero dispatch result |
| Input entry at 0x14 | Select default filtering mode | Low 32-bit return word selects disabled/enabled filtering |
| Recipient entry at 0x1c | Obtain successor | Low 32-bit return word supplies the next recipient address |
| Input entry at 0x04 | Final input operation | Called once for a nonnull input on a normal-return path |

No additional semantics, global-state preservation, or memory-ownership guarantees are established for these dependencies.

## Acceptance examples

These are conditional expectations derived from static evidence, not executed tests. Each assumes valid required memory, returning dependencies, and no unspecified mutation except where stated.

1. **Collection recipient handles the input.**  
   Enumeration produces one recipient returning zero and then one returning 7. Both are invoked in order. Collection cleanup occurs once. No explicit or default recipient is invoked. A nonnull input receives its final operation, and the function returns 7.

2. **Any nonzero result terminates dispatch.**  
   A collection recipient returns the 32-bit pattern 0xffffffff. Collection enumeration stops and cleanup occurs. The function returns that same pattern; it is not normalized to one or rejected as negative.

3. **Empty collection with an explicit recipient.**  
   Collection setup occurs, the first enumeration call returns zero, and collection cleanup occurs. The explicit recipient returns zero. The function returns zero without default traversal.

4. **Collection absent and input null.**  
   The first context’s collection field is zero, no explicit recipient exists, and the second context supplies a default head. No input classification or eligibility dependency is invoked. Default recipients are offered the null input until one returns nonzero or traversal ends. No final input operation occurs.

5. **Filtering selectively skips recipients.**  
   Classification has a nonzero low word. A default recipient has a valid associated record with a nonzero filtering field, and F-35b8bd5fa9e0 returns zero. Its handler is skipped. If its associated-record field remains nonzero, its successor operation is still called.

6. **Zero filtering field bypasses the eligibility dependency.**  
   Filtering is enabled, but the current recipient’s associated-record filtering field is zero. F-35b8bd5fa9e0 is not called for that recipient. Its handler is invoked.

7. **A zero handler result can terminate traversal.**  
   Filtering is disabled. The current handler returns zero and leaves the recipient’s associated-record field zero. Traversal ends without calling its successor operation.

8. **A handler can change advancement.**  
   Filtering is enabled and the associated record is initially valid. The eligible handler returns zero after setting the recipient’s associated-record field to zero. The subsequent fresh read ends traversal; no successor operation follows.

9. **Upper result words do not select behavior.**  
   Classification returns a zero low word and a nonzero upper word. Filtering remains disabled. Separately, a successor result with a zero low word ends traversal even if its upper word is nonzero.

10. **Null associated record is not generally safe under filtering.**  
    Filtering is enabled and a current recipient has a null associated-record address. The function attempts the associated-record filtering-field read before the later termination check. A clean zero return is not guaranteed.

## Uncertainties

The following are **documented_only**:

- The application-level meanings of input objects, recipients, contexts, collection values, and filtering fields.
- The semantic meaning of the setup bit pattern 0xfffffffe.
- Collection enumeration order and behavior under collection mutation.
- The ownership effects of every dependency and virtual operation.
- Whether the final input operation releases, destroys, or merely notifies the input.
- The meanings of upper words returned by classification and successor operations.
- Thread-safety, reentrancy, synchronization, and concurrent-mutation guarantees.
- Failure behavior inside dependencies, including exceptions or nonlocal exits.
- Whether malformed layouts or cyclic traversals are excluded by wider-system invariants.

“Handled,” “classification,” “eligibility,” “cleanup,” and “successor” describe observed control-flow roles. They are inferred intent labels, not established application semantics.

## Evidence

Established static evidence consists of the supplied function body and the opaque dependency mapping. It supports the field offsets, 32-bit tests, virtual-entry offsets, dependency call order, collection working-area size, dispatch precedence, traversal rules, and final input-operation ordering described above.

The contract distinguishes visible operations from unknown callee behavior. It does not assume that dependencies preserve globals, preserve object lifetimes, or own or release memory.

No implementation access, runtime execution, or tests were used. The acceptance examples are static conditional predictions, not observed results.
