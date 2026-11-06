<!-- Trusted provenance: F-cff93e792fdd; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-cff93e792fdd

## Purpose

F-cff93e792fdd coordinates temporary-storage acquisition, preparation, repeated polling and scanning, conditional result allocation, result processing, and cleanup. It returns a 32-bit status. On a qualifying scan result, it directly writes an allocation-like token into a caller-provided output slot before invoking result processing.

The apparent intent is to find and construct a result. That interpretation is inferred; the supplied body does not establish the result’s format, the allocation units, or the semantic meanings of the status and state values.

This contract distinguishes the function’s direct actions from dependency effects. A dependency may modify storage through an independently available alias or an address retained from an earlier call. Consequently, the absence of a direct output-slot write does not guarantee unchanged observable slot contents, and a value written before a dependency call need not remain unchanged afterward.

## Interface

### Inputs and return value

| Input | Established representation | Direct use |
|---|---|---|
| First input | Opaque 32-bit value | Passed to preparation, polling, scanning, and processing. |
| Second input | Opaque 32-bit value | Passed to preparation and scanning. |
| Output slot | Address of a writable 32-bit word | Receives the result-allocation token whenever publication is reached. |
| Fourth input | Opaque 32-bit value | Passed only to processing. |

The return value is a signed 32-bit status in the supplied representation.

The first, second, and fourth inputs are not directly dereferenced by this function. Their semantic types and valid domains are **documented_only**.

The function does not directly read or initialize the output slot. Its direct writes to that slot occur only on the result-publication path. A writable slot is required when that path is reached; there is no local validity or null check. Paths without publication perform no direct slot access, although dependencies may have independent access to the same storage.

### Internal values exposed to dependencies

The function uses:

- A 32-bit temporary-allocation token.
- Three opaque 32-bit preparation values.
- Two signed 32-bit scan values.
- A signed 32-bit state word.
- A signed 32-bit current status.

The preparation values are identified below as **first**, **second**, and **third**, according to the order of their output-address arguments. The scan values are likewise identified by their output-address order. These names describe argument relationships, not a public structure or contiguous layout.

After successful initial acquisition, the state is initialized to zero before preparation. The preparation and scan output words have no visible local initialization before their addresses are supplied to the corresponding dependencies. Dependencies must establish any subsequently consumed values for those reads to have a defined, meaningful interpretation.

The internal addresses are valid only within the invocation’s storage lifetime. Passing them to dependencies does not establish permission to use them after the invocation ends.

The supplied representation establishes 32-bit data words and status arithmetic. It does not independently establish a public buffer layout, allocation unit, alignment requirement, or pointer width.

## Behavior

### Initial acquisition and preparation

The first dependency call is F-d33f6d597e5d with request value `512`.

If the returned token is zero:

- The function returns `1`.
- No other dependency is called.
- The function performs no direct output-slot write.

The initial allocator itself could affect independently reachable storage. Unchanged observable output-slot contents therefore require that dependency to preserve the slot.

If the token is nonzero, the function initializes the state to zero and calls F-4008d9582460 with:

1. The first input.
2. The second input.
3. The first preparation value’s address.
4. The second preparation value’s address.
5. The third preparation value’s address.

The preparation return becomes the current status. A nonzero status leads directly to cleanup. A zero status permits polling.

The temporary allocation is not directly inspected or initialized by this function.

### Polling and scan eligibility

Each polling iteration first calls F-e27353dd9a60 with the first input. The state does not suppress this call.

The subsequent decision has the following order:

- A nonzero polling return leads to cleanup without a state read for scan eligibility.
- A zero polling return is followed by a state read.
- A nonzero state leads to cleanup.
- A zero state permits scanning.

The polling return does not replace the current status. A polling return of `9`, for example, does not itself cause a final return of `9`.

The relevant state is the value observed **after polling returns**. Once scanning has received the state address, a later dependency could modify that state through a retained address. The function does not establish that polling preserves it.

### Scanning

An eligible scan calls F-fd20b5807437 with:

1. The first input.
2. The second input.
3. The second preparation value.
4. The first scan value’s address.
5. The second scan value’s address.
6. The temporary token.
7. The state address.

The scan return replaces the current status.

A nonzero scan status causes immediate cleanup. No result allocation occurs on that path, even if scanning also sets the state to `1`.

After a zero scan return, the function reads the state:

- Exactly `1` selects result allocation and processing.
- Any other value returns control to polling.

A successful scan leaving state `2` therefore causes another poll, not immediate cleanup. If that poll returns zero and leaves state nonzero, cleanup follows. If it returns zero after changing state to zero, another scan follows. A nonzero poll return selects cleanup regardless of the state it leaves.

Similarly, a successful scan leaving state zero does not guarantee another scan: polling intervenes and can stop the operation.

There is no local iteration bound or timeout.

### Result allocation and arithmetic

A zero scan status followed by state exactly `1` causes another call to F-d33f6d597e5d.

The request is formed by adding the second scan value to the first scan value, then adding `20`. The supplied representation uses signed 32-bit operands and additions.

There are no explicit checks for negative operands, either addition overflowing, an unsuitable resulting request, or failure of this allocation request.

For ordinary signed mathematical evaluation, both the intermediate sum and the final sum must be within −2,147,483,648 through 2,147,483,647. A representable final mathematical value is insufficient if the first addition overflows. Behavior outside those bounds, and any request conversion or interpretation at the dependency boundary, remain **documented_only**. Modular wrapping is not an established guarantee.

The literal addition of `20` is established. Its interpretation as a header or other storage overhead is not.

### Publication and processing

After result allocation returns, the function directly writes its 32-bit return token into the output slot. It publishes zero as well as nonzero tokens; there is no separate result-allocation failure branch.

Publication precedes the processing call. The function does not recheck state between selecting this result path and invoking processing. A state change during allocation does not by itself cancel publication or processing.

F-a18dd8c464d0 receives:

1. The first input.
2. The first preparation value.
3. The third preparation value.
4. The first scan value.
5. The second scan value.
6. The temporary token.
7. The second preparation value.
8. The result-allocation token just published.
9. The fourth input.

The processing return replaces the current status.

A nonzero processing status leads to cleanup. A zero processing status returns control to polling; it does not immediately return success.

The function itself does not reset state after processing. If dependencies preserve state `1`, successful processing is followed by one poll and then cleanup: a nonzero polling return stops immediately, while a zero polling return exposes the preserved nonzero state.

That preservation is conditional. If polling instead returns zero with state zero, another scan occurs and may lead to another allocation and publication. The body therefore establishes no unconditional one-publication limit.

Preparation and scan values may also change through retained addresses. In particular, the scan values passed to processing need not equal the values previously used to calculate the allocation request if allocation modifies those words through retained addresses. The token passed to processing is the allocation return retained by this function, not a reread of the caller’s output slot.

The body does not establish a relative read order among the individual value arguments of a dependency call.

### Cleanup and return selection

Every normally returning path after a nonzero initial allocation calls F-5e3309c9d083 exactly once with the temporary token. Any cleanup return value is ignored.

Final status selection occurs after cleanup:

- A nonzero current status is returned unchanged.
- A zero current status causes a state read.
- State exactly `1` yields return `0`.
- Every other state yields return `7`.

A preparation, scan, or processing error therefore takes precedence over final state. A polling return does not, because it is not assigned to the current status.

Cleanup can affect the zero-status result if it modifies state through a previously exposed address. The decisive state read occurs after cleanup, not before it. For a nonzero current status, this final decision does not require a state read.

These cleanup guarantees concern normal returns. The body supplies no recovery contract for dependencies that fail to return normally.

## Side effects

The directly established effects are internal assignments, dependency calls, conditional 32-bit output-slot writes, and a returned status.

The function performs no direct output-slot clearing on entry, on an early failure, or after publication. It performs no direct rollback of a published token when processing fails. These are statements about this function’s writes, not guarantees that dependencies preserve observable slot contents.

A later publication directly overwrites whatever value then occupies the output slot. The function makes no direct release call for an earlier result token. Dependencies could nevertheless release, replace, or otherwise affect such resources under contracts not supplied here.

The temporary token is passed to the cleanup dependency. This establishes a cleanup call, not proof of memory release. No allocation ownership transfer, result validity, or post-error usability guarantee follows solely from publication.

No direct global-variable access is visible. Dependencies may modify globals, perform I/O, retain addresses, or access the output slot through independent aliases. Such behavior is neither required nor ruled out by this body.

## Dependencies

| Opaque ID | Established caller-facing role |
|---|---|
| F-d33f6d597e5d | Receives `512` initially and computed requests conditionally; returns a 32-bit allocation-like token. Only the initial return is checked for zero. |
| F-4008d9582460 | Receives two inputs and three preparation-output addresses; returns the preparation status. |
| F-e27353dd9a60 | Receives the first input before each scan-eligibility decision; its return controls stopping without replacing current status. |
| F-fd20b5807437 | Receives inputs, preparation data, temporary storage, two scan-output addresses, and the state address; returns the scan status. |
| F-a18dd8c464d0 | Receives selected inputs, preparation and scan values, and temporary and result tokens; returns the processing status. |
| F-5e3309c9d083 | Receives the temporary token during normal cleanup; its return is ignored. |

These role descriptions do not supply missing dependency contracts, including preservation, ownership, allocation, or release semantics.

## Acceptance examples

These examples are conditional static trace requirements, not executed tests. Dependencies are assumed to return normally, and consumed output words are assumed to have been established.

1. **Initial acquisition fails.**  
   The `512` request returns zero. The function returns `1`, makes no further calls, and performs no direct output-slot write. The observable slot remains unchanged only if the allocator preserves it through all independently available aliases.

2. **Preparation fails.**  
   Initial acquisition succeeds and preparation returns `12`. Cleanup receives the temporary token, then the function returns `12`. No polling, scanning, or direct publication occurs.

3. **Polling stops before scanning.**  
   Preparation returns zero. Polling returns `9`. If state remains zero through cleanup, the function returns `7`, not `9`. No scan occurs.

4. **Scan failure takes precedence.**  
   An eligible scan returns `5` and writes state `1`. Cleanup follows immediately, and the function returns `5`. No result allocation occurs.

5. **A non-result state is preserved.**  
   A successful scan writes state `2`. Another poll occurs. If that poll returns zero and preserves state `2`, cleanup follows without another scan. If cleanup also preserves state `2`, the return is `7`.

6. **Polling enables another scan.**  
   A successful scan writes state `2`. The next poll changes the retained state word to zero and returns zero. Another scan follows.

7. **Polling stops despite resetting state.**  
   After a successful scan, polling changes state to zero but returns `9`. Cleanup follows without another scan. If cleanup preserves zero, the function returns `7`.

8. **Successful result with preserved state.**  
   A successful scan establishes first scan value `100`, second scan value `24`, and state `1`. If those values remain unchanged until request evaluation, the allocation request is `144`. The returned token is directly published before processing. If processing returns zero and subsequent dependencies preserve state `1`, one more poll occurs, then cleanup, and the function returns zero.

9. **Result allocation returns zero.**  
   The result request returns zero. The function directly writes zero to the output slot and invokes processing with zero as its result-token argument. No dedicated local allocation-failure status is introduced. Dependencies may subsequently alter the slot through independent aliases.

10. **Processing fails after publication.**  
    A nonzero token is directly published, and processing returns `6`. Cleanup follows, then the function returns `6`. The function performs no direct clearing or rollback. The slot still contains that token only if processing, cleanup, and any other applicable dependency effects preserve it.

11. **Repeated publication is reachable.**  
    Following successful processing, polling changes state to zero and returns zero. A later successful scan establishes state `1`. Another allocation and direct publication follow. This function does not directly release the earlier token.

12. **Cleanup changes success selection.**  
    After a successful scan has exposed the state address, polling selects cleanup while current status is zero. Cleanup changes state to `1`. The final return is zero because the decisive read follows cleanup.

13. **Negative request without overflow.**  
    If the scan values used in request evaluation are `−30` and `0`, the request expression evaluates to `−10` without signed overflow. The function has no local rejection branch for that value; its meaning to the allocation dependency remains unspecified.

## Uncertainties

The following remain **documented_only**:

- Semantic input types, valid domains, and status meanings.
- State meanings beyond comparisons with zero and exactly `1`.
- Storage layouts, allocation units, alignment, and the meaning of `20`.
- Valid scan-value ranges, signed-overflow behavior, and request-boundary conversions.
- Dependency guarantees for initializing output words.
- Ownership, release behavior, token validity, and result usability after errors.
- Independently reachable aliases to the caller’s output slot.
- Retained-address mutation, reentrancy, concurrency, and global effects.
- Behavior for invalid addresses, uninitialized consumed values, or dependencies that do not return normally.

These unknowns do not erase the established direct ordering: polling precedes scan eligibility, publication precedes processing, and cleanup precedes the final zero-status state test.

## Evidence

Established behavior derives from static examination of the supplied function body: represented widths, explicit assignments, dependency arguments, branch conditions, arithmetic grouping, the output-slot store, and cleanup ordering.

The distinction between direct output-slot writes and observable slot contents follows from the absence of dependency contracts excluding independent aliases. Likewise, exposing internal addresses permits consideration of later dependency mutation without establishing that such mutation actually occurs.

The scan-and-construct purpose is inferred intent. Dependency internals, resource ownership, and unshown layouts remain unresolved.

No runtime tests have been performed.
