<!-- Trusted provenance: F-a7ae6deda11a; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-a7ae6deda11a

## Purpose

F-a7ae6deda11a conditionally prepares and attaches an object to a destination context, checks a state field in the attachment, and invokes a subsequent operation on the context. If that operation returns zero, the function may invoke an embedded-member update using a value read from the input.

The visible control flow distinguishes three main outcomes:

- A prerequisite check returns zero: return literal zero without entering the construction path.
- The attachment is null after temporary cleanup, or its designated state field is all ones: write the 32-bit error representation `0xfffffff9`, conventionally interpreted as signed −7.
- A non-null attachment with any other state value reaches F-eed279d2c535: store its 32-bit result as context status and conditionally update the embedded member.

A zero return does not establish that construction occurred or that F-eed279d2c535 ran. Conversely, the invalid-state path does not invariably return −7, because an intervening indirect call can change the status subsequently returned.

“Context,” “attachment,” “construction,” and “cleanup” describe apparent roles. Their application-level meanings and ownership implications are not established.

## Interface

### Inputs

The function receives two address-like arguments:

1. **Destination context address.** Used for direct status and attachment writes, direct reads, and formation of an embedded-member address.
2. **Input address.** Passed to dependencies, compared with the embedded-member address, and conditionally used for a 32-bit read at offset +4.

The supplied representation uses 32-bit argument and address-like values. It does not establish equivalent behavior under a wider-pointer interface.

The function also reads a **global data slot** on the construction path. That slot supplies a 32-bit value copied through an address returned by F-b9acea4b06ad. It is an implicit input, not an established constant.

### Return value

The return is 32 bits.

| Path | Returned value |
|---|---|
| Prerequisite check returns zero | Literal zero |
| Attachment is null after temporary cleanup | Literal `0xfffffff9` |
| Attachment has the all-ones state | Context status read after the indirect call and attachment clearing |
| F-eed279d2c535 runs | Context status read after the possible embedded-member update |

The final status read matters. Dependencies called after a status write are not established to preserve that field.

### Visible storage layout

All offsets are in bytes.

| Storage | Offset | Width | Visible use |
|---|---:|---:|---|
| Destination context | +4 | 32 bits | Status |
| Destination context | +8 | 32 bits | Attachment address |
| Destination context | +12 | Complete extent unknown | Embedded-member address |
| Destination context | +16 | 32 bits | Prerequisite-check argument |
| Input | +4 | 32 bits | Conditional update argument |
| Attachment | +0 | 32 bits | Indirect-call table address |
| Attachment | +24 | 32 bits | State compared with `0xffffffff` |
| Indirect-call table | +4 | 32 bits in the visible representation | Call target |
| Local temporary | — | 8 bytes | Storage passed to three dependencies |
| Global data slot | — | 32 bits | Value read and copied during preparation |
| Address returned by F-b9acea4b06ad | +0 | 32 bits | Destination of that copy |

The complete context, input, attachment, and embedded-member layouts are unknown. In particular, the embedded-member address at context +12 does not establish that the member occupies only four bytes.

### Preconditions

For ordinary execution of the taken path:

- Directly accessed context fields must support the indicated reads and writes.
- The global data slot must permit a 32-bit read when construction preparation reaches it.
- F-b9acea4b06ad must return an address supporting a 32-bit write.
- A non-null attachment examined by this function must support a 32-bit read at +24.
- On the invalid-state path, the attachment’s first word and the table entry at +4 must permit resolution and invocation of the indirect target.
- On the conditional update path, input +4 must support a 32-bit read.
- Dependencies must accept their supplied arguments.

These conditions are not visibly validated. Invalid addresses, inaccessible storage, malformed tables, or invalid dependency results have no established recoverable-error behavior.

## Behavior

### Initial status reset and prerequisite check

The first visible memory effect is a 32-bit zero write to context +4.

F-44362605ab79 is then called with the input address. After that call returns, the function reads the 32-bit context field at +16 and calls F-4a6c0bf0740c with:

- The value just read from context +16.
- The result returned by F-44362605ab79.

The ordering is significant. Changes to context +16 made during F-44362605ab79 can affect the argument supplied to F-4a6c0bf0740c.

Only zero versus nonzero is tested in the prerequisite result. No distinction among nonzero values is established.

If the result is zero, the function returns literal zero. On this path, it does not directly:

- Prepare the local temporary or read the global data slot.
- Construct or overwrite an attachment.
- Invoke temporary cleanup or the indirect target.
- Invoke F-eed279d2c535 or F-151ab4bfe553.

It also does not restore status after either prerequisite dependency. Thus the return remains zero even if a prerequisite dependency changes context status to a nonzero value. An existing attachment is not directly cleared or replaced on this path, although dependency effects remain unknown.

### Preparation, global-value transfer, and construction

For a nonzero prerequisite result, the visible sequence is:

1. Call F-8157a180d901 with the fixed argument `0x54`, or 84 decimal.
2. Call F-b9acea4b06ad with the address of the 8-byte local temporary and the input address.
3. After F-b9acea4b06ad returns, read the 32-bit global data slot.
4. Write the value just read through the address returned by F-b9acea4b06ad.
5. Call F-6dc622709f3d with seven arguments, in this order: the result from F-8157a180d901, the temporary’s address, `1`, `0`, `0x400`, `1`, and `0`.
6. Write the construction result to context +8.
7. Call F-289ec4311c64 with the temporary’s address.
8. Reload the attachment address from context +8.

The global read is not a use of an established fixed value. Its contents are read at this point in the sequence, after both preparation calls. Any preceding dependency that changes that slot can affect the value copied. Neither its immutability nor preservation by the preceding dependencies is established.

The global read precedes the store through the returned address. The relationship between that destination, the temporary, the global slot, and other reachable storage is not established. A distinct destination must not be assumed.

The result from F-8157a180d901 is not checked before being passed to F-6dc622709f3d. A zero result from F-8157a180d901 therefore does not itself select the explicit −7 path.

The construction result is stored before temporary cleanup. Attachment validation uses the field reloaded after F-289ec4311c64 returns, not necessarily the original construction result. If that dependency changes context +8, the changed value controls the next branch.

### Null attachment

If the reloaded attachment address is zero, the function writes `0xfffffff9` to context +4 and returns the same literal 32-bit value.

No dependency call intervenes between that status write and return in the visible flow. This branch does not invoke the indirect target, F-eed279d2c535, or F-151ab4bfe553.

### Invalid attachment state

For a non-null attachment, the function reads its 32-bit field at +24. Exactly `0xffffffff`, the all-ones representation conventionally interpreted as signed −1, selects the invalid-state path.

That path performs these effects in order:

- Write `0xfffffff9` to context +4.
- Read the attachment’s first word to locate an indirect-call table.
- Obtain the call target from table offset +4 and invoke it.
- After normal return, write zero to context +8.
- Read context +4 and return its value.

The indirect call occurs before the attachment field is cleared. If the target writes another attachment address into context +8, the subsequent direct zero write overwrites that value.

Status is not reset to −7 after the call. If the indirect target changes context +4, that changed status is returned.

The visible call expression supplies no explicit arguments. Receiver handling and the complete calling convention are not established. Its apparent cleanup role does not prove that it destroys the attachment or releases any resource.

### Subsequent operation and conditional update

Every attachment state other than `0xffffffff` reaches F-eed279d2c535, called with the context address.

The function writes that dependency’s 32-bit return value to context +4. This write follows the call and therefore overwrites any earlier status value left by that call.

The decision to invoke the embedded-member update uses the returned result itself:

- A nonzero result skips the update.
- A zero result permits the update only when the input address differs exactly from context +12.

This is address equality, not content equality or a general overlap check. Distinct addresses can still refer to overlapping storage.

If the update is permitted, the function reads a 32-bit value from input +4 and calls F-151ab4bfe553 with the embedded-member address at context +12 and that value.

The input read occurs after F-eed279d2c535 returns and after its result is written to status. Consequently, dependency mutations or permitted aliasing can affect the value read. No pre-operation snapshot of input +4 is established.

The function finally reads context +4 and returns it. F-151ab4bfe553’s return value, if any, is not visibly used; any status mutation it makes can affect the final return.

There is no direct attachment clearing or additional cleanup in response to a nonzero result from F-eed279d2c535.

## Side effects

### Direct memory effects

Depending on the path, the function directly:

- Writes zero to status.
- Reads context +16 for the prerequisite check.
- Reads the global data slot and copies its current 32-bit value through a dependency-returned address.
- Replaces context +8 with the construction result.
- Reloads that field after temporary cleanup.
- Reads attachment state and, on the invalid-state path, indirect-call metadata.
- Writes −7 or F-eed279d2c535’s result to status.
- Clears context +8 after the invalid-state indirect call.
- Reads input +4 for the conditional update.
- Reads status for the final return on non-null attachment paths.

The transfer from the global slot is not intrinsically a write to the temporary: the destination is whatever address F-b9acea4b06ad returns. Its actual aliasing relationships remain unknown.

There is no direct release or preservation of a pre-existing attachment before the construction result overwrites context +8. Safety therefore depends on external invariants or dependency behavior not supplied here.

The embedded-member update is delegated to F-151ab4bfe553; the function does not directly assign the member at +12.

### Ownership and lifetime

The local temporary occupies eight bytes and is passed to F-b9acea4b06ad, F-6dc622709f3d, and F-289ec4311c64. No direct initialization of its contents is visible before the first of those calls.

F-289ec4311c64 is called once on the normal construction path, before attachment validation.

The evidence does not establish:

- Whether F-8157a180d901 allocates storage.
- Whether F-6dc622709f3d borrows, consumes, or replaces its first argument.
- Whether temporary cleanup releases resources.
- Whether the indirect target destroys or releases the attachment.
- Whether the context owns a successful attachment.
- Whether a dependency retains a pointer to the temporary.

No ownership transfer or lifetime guarantee follows solely from call order.

### Errors and interruption

The only explicit error representation written by this function is `0xfffffff9`. Other nonzero results from F-eed279d2c535 are stored without normalization.

No visible exception handler, retry, rollback, or cleanup guard handles a dependency that fails to return normally. Earlier effects may already have occurred. Recovery and unwinding behavior are unknown.

## Dependencies

| Opaque ID | Established interaction |
|---|---|
| F-44362605ab79 | Receives the input address; its result becomes the prerequisite check’s second argument. |
| F-4a6c0bf0740c | Receives the post-call context +16 value and the preceding result; zero skips construction. |
| F-8157a180d901 | Receives 84; its unchecked result becomes the first construction argument. |
| F-b9acea4b06ad | Receives the temporary and input addresses; returns the destination for a 32-bit store performed after reading the global slot. |
| F-6dc622709f3d | Receives the seven construction arguments; its result is stored as the attachment. |
| F-289ec4311c64 | Receives the temporary address; attachment reloading and validation follow its return. |
| F-eed279d2c535 | Receives the context address; its result is stored as status and gates the update. |
| F-151ab4bfe553 | Receives context +12 and the conditionally read input word. |

An additional indirect dependency is selected through the invalid attachment’s table entry at +4. No opaque identity is supplied for it.

No dependency is assumed to preserve the global slot, context fields, input storage, or other reachable memory. Dependency-specific error and ownership contracts are unavailable.

## Acceptance examples

These are conditional acceptance scenarios, not executed tests. Each assumes normal dependency returns and accessible storage for the selected path.

1. **Prerequisite rejection with unchanged storage.**  
   If the prerequisite result is zero and the prerequisite dependencies preserve context storage, the return and status are zero, and an existing attachment remains unchanged. Construction preparation and the global read do not occur.

2. **Prerequisite rejection after a status change.**  
   If a prerequisite dependency changes status to another 32-bit value and the check returns zero, the function returns literal zero without restoring status.

3. **Preparation changes the global value.**  
   Suppose the global slot initially contains `0x11223344`, and F-b9acea4b06ad changes it to `0xaabbccdd` before returning a valid, distinct destination address. If no intervening modification occurs, the subsequent direct store writes `0xaabbccdd`, not the initial value.

4. **Null attachment after cleanup.**  
   If context +8 is zero after F-289ec4311c64 returns, the function writes and returns `0xfffffff9`. The original construction result does not override that post-cleanup observation.

5. **Cleanup replaces the attachment.**  
   If construction stores one address and temporary cleanup replaces it with another valid address, the replacement’s state at +24 determines the next branch.

6. **Invalid state with preserved status.**  
   If attachment state is `0xffffffff` and the indirect target preserves status, the final attachment field is zero and the return is `0xfffffff9`.

7. **Invalid state with changed status and attachment.**  
   If the indirect target writes status `9` and a nonzero attachment address, the function subsequently clears the attachment and returns `9`.

8. **Nonzero subsequent-operation result.**  
   If F-eed279d2c535 returns `5`, the function stores and returns `5`, skips F-151ab4bfe553, and performs no direct attachment cleanup on that branch.

9. **Successful exact self-aliasing.**  
   If F-eed279d2c535 returns zero and the input address equals context +12, the update and its direct input +4 read are skipped.

10. **Successful distinct-address update.**  
    If the result is zero and the addresses differ, the update receives the word at input +4 as read after the operation and status write. If F-151ab4bfe553 then changes status to `3`, the function returns `3`.

## Uncertainties

The following remain **documented_only**:

- Concrete types, alignment requirements, byte order, and complete layouts.
- The global slot’s meaning, initialization, mutability rules, and possible aliases.
- The meaning of 84 and the construction arguments, including `0x400`.
- Dependency side effects, failure conventions, ownership transfers, and nonlocal exits.
- The returned store address’s relationship to the temporary and other storage.
- The indirect target’s identity, receiver convention, and release semantics.
- Required initial state of the attachment and embedded member.
- Synchronization, reentrancy, and concurrent modification behavior.
- Address-arithmetic behavior outside valid objects or the target address range.

The state comparison is an exact 32-bit all-ones check. The error value is an exact 32-bit representation; its application-level meaning is unknown. No overflow calculation is needed for the fixed numeric arguments themselves. Validity of address additions is still required; wraparound is not an established supported case.

## Evidence

**Established static evidence:** the supplied representation supports the stated field widths and offsets, constants, dependency arguments, branch conditions, global read followed by a store through the returned address, and ordering of direct memory accesses and calls.

**Inferred intent:** preparation, construction, attachment readiness, cleanup, and embedded-member update are descriptive interpretations of the visible pattern, not verified dependency semantics.

**Unknown:** dependency implementations, global-slot semantics, ownership rules, complete calling conventions, and runtime behavior beyond the visible operations.

No dependency implementations or runtime observations were supplied. No runtime tests have been performed.
