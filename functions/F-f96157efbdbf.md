<!-- Trusted provenance: F-f96157efbdbf; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-f96157efbdbf

## Purpose

F-f96157efbdbf conditionally decrements a 32-bit field associated with the address held in a caller-supplied slot. It distinguishes a zero decrement result from a nonzero result, invokes dependencies in a defined order, and clears the caller’s slot on nonempty paths that reach its final store.

The zero-result path may invoke an indirect callback. Subsequent processing may reach a dependency annotated as non-returning; those paths do not ordinarily reach the slot-clearing store or return to the caller.

The apparent intent is reference release followed, when the count reaches zero, by finalization and storage-related processing. This is **inferred intent**, not an established ownership contract. The supplied evidence does not establish whether any dependency acquires, releases, destroys, or deallocates anything.

## Interface

### Input and output

The function accepts the address of a slot containing a 32-bit value interpreted either as zero or as an object address.

On every ordinary return, the return value is the original slot address. It is not the object address, the decremented value, or a status code.

The function immediately reads through its argument. There is no explicit check that the slot address itself is nonzero or valid. A valid slot containing zero is supported; an invalid slot address is not a checked error case.

### Widths and accessed layout

The supplied representation uses four-byte integers and four-byte address-bearing values. The visible object accesses are:

| Byte offset | Accessed value |
|---|---|
| 0 | Address used to begin callback dispatch, or zero |
| 4 | Integer read, decremented, and written |
| 8 | Value tested before the decrement and during later terminal selection |

For callback dispatch, the offset-0 value identifies a structure whose first four-byte entry supplies another address. The callback address is read at byte offset 4 from that second address.

These observations define access requirements, not a complete object type. They do not establish additional fields, allocation size, alignment requirements, or the semantic type of the offset-8 value.

The decrement is represented as 32-bit integer arithmetic. For representable results, it subtracts one exactly. Decrementing the minimum signed 32-bit value is outside that representable range; the supplied evidence does not establish overflow behavior. Wraparound is not an unconditional part of this contract.

### Preconditions for fault-free progress

The slot must support the initial read. Every later access requires its then-used address to remain valid for that access. In particular:

- Reaching the final store requires the original slot to remain writable.
- Reaching the decrement requires a valid object address in the slot after any preliminary dependency returns.
- Reaching callback dispatch requires a readable dispatch chain and a callable resolved address.
- Reaching a post-dependency object access requires the relevant object storage still to exist and remain readable.
- Continued execution after a dependency requires that dependency to return normally.

These are conditional access requirements, not validations performed by the function. No dependency is assumed to preserve the slot, object fields, dispatch structures, global state, or storage lifetime.

## Behavior

### Initially empty slot

The first operation reads the caller’s slot.

If that read yields zero, the function returns the original slot address. It does not access object fields, call dependencies, invoke a callback, or write zero back to the slot.

### Preliminary operation for a nonempty slot

If the initial slot read is nonzero, the function reads the offset-8 field through a slot-derived object address.

If that field is nonzero, it calls F-338d16f9ce72 with no explicit arguments shown. If the field is zero, it skips this call.

The slot is read again for the later decrement. The object selected after F-338d16f9ce72 need not be the object present at entry. For example, that dependency may replace the slot, alter the selected object’s count, or change its dispatch field. Normal return alone does not imply preservation of any of those values.

If the dependency clears the slot, the function does not perform another nonzero check before the count access. Such a change therefore does not produce the initially-empty no-op behavior.

The meaning of the offset-8 condition is unknown. Its use does not establish a lock, allocator, ownership mode, or allocation flag.

### Decrement and branch selection

After the optional preliminary call, the function reads the 32-bit integer at offset 4 through the current slot value, subtracts one, and stores the computed result at offset 4 through a slot-derived address.

The count read and count write use separate slot-derived accesses. No external call appears between them. Nevertheless, the evidence does not establish atomicity or protection against concurrent mutation.

The branch is selected using the computed decrement result, not by rereading the stored count. Thus later modification of the count does not retroactively change which branch this computation selected.

There is no positive-count validation:

- A count of 1 produces zero and selects the zero-result branch.
- A count of 0 produces −1 and selects the nonzero-result branch.
- Other counts with representable nonzero decrement results also select the nonzero-result branch.

If the slot aliases an accessed object field, a direct write may itself affect subsequent slot-derived accesses. Such aliasing is not explicitly excluded or handled.

### Nonzero-result branch

When the computed result is nonzero, the function calls F-0923d65e0f4b with no explicit argument shown.

If this call returns normally, execution reaches the final store: zero is written to the original caller-supplied slot, and the original slot address is returned.

The final store follows the dependency call. A replacement value written into the slot by that dependency is overwritten, provided the slot remains writable and execution reaches the store.

This branch does not invoke the indirect callback, F-a600efb7b035, or F-feee9156a257.

### Zero-result branch: callback selection and dispatch

When the computed result is zero, the function reads offset 0 of the object then designated by the slot.

If this field is zero, it skips the callback. If it is nonzero, the function resolves the callback by:

- Using the object’s offset-0 value as an address.
- Reading the first address-bearing entry there.
- Reading the callback address at offset 4 from the resulting address.

The callback is invoked with no explicit arguments shown. Its receiver convention, effective signature, return-value meaning, and ownership effects are not established.

The eligibility check and dispatch resolution involve separate accesses. They are not established as a single immutable snapshot. A nonzero eligibility field also does not validate the rest of the dispatch chain.

The relevant count and dispatch conditions occur after the optional preliminary operation. An entry count of 1 or an entry nonzero dispatch field alone is insufficient to guarantee this callback path when F-338d16f9ce72 runs.

### Zero-result branch: common dependency

After the callback returns, or immediately if it was skipped, the function reads the current slot value and supplies that value explicitly to F-0923d65e0f4b.

This differs from the nonzero-result branch, which shows no explicit argument for the same dependency. The supplied representation does not establish a consistent shared prototype or justify inventing missing arguments.

The explicit value on this branch need not equal the entry object address or the object used for the decrement. In particular, a callback that replaces or clears the slot changes the value supplied to this dependency.

After F-0923d65e0f4b returns, the function reads the slot again. This newly read value is retained for subsequent selection and object access.

### Zero-result branch: terminal selection

If the retained post-call slot value is zero, execution reaches the final zero store and returns the original slot address.

If it is nonzero, the function reads offset 8 from the retained object:

- If that field is zero, F-feee9156a257 is called with the retained object address.
- If that field is nonzero, F-a600efb7b035 is called with no explicit arguments shown. After it returns, offset 8 is read again from the retained object, and this fresh field value is supplied to F-feee9156a257.

The repeated field read is observable. F-a600efb7b035 may change the offset-8 value, including changing it to zero. The terminal argument is the value obtained by the later read, not necessarily the value that selected this path.

The retained object address is not refreshed after F-a600efb7b035. Replacing the caller’s slot during that dependency does not redirect the final field read. Invalidating the retained object can instead make that read invalid.

Both visible F-feee9156a257 calls are annotated as non-returning. Under that annotation, neither path reaches the final slot-clearing store or an ordinary return. What causes the dependency not to return is unknown.

### Ordinary-return postcondition

Every visible ordinary return yields the original argument address.

For an initially empty slot, no store is performed. For a nonempty path reaching the final store, the function writes zero to the original slot after the preceding dependency operations.

This is not a guarantee that all invocations return or clear the slot. Faults, nonlocal exits, and annotated terminal calls may prevent completion.

## Side effects

Established direct effects are:

- Repeated reads of the caller’s slot.
- Conditional reads of object fields at offsets 0, 4, and 8.
- Conditional reads through the callback dispatch chain.
- One count-field write on an execution reaching the decrement.
- A final zero write on a nonempty path reaching ordinary completion.
- Calls to the dependencies and optional callback in the order described above.

There is no visible rollback of the count update if a later operation fails or does not return. There is no explicit error result, recovery handler, allocation operation, or deallocation operation in the supplied representation.

The slot is not cleared before the callback and dependency calls. Reentrant code may observe a nonzero slot associated with an already-decremented count, including zero. Neither reentrancy safety nor concurrent-access safety is established.

External effects remain unknown. Calling a dependency with no explicit arguments does not establish that it has no inputs or cannot affect reachable memory.

## Dependencies

| Dependency | Visible invocation | Established position |
|---|---|---|
| F-338d16f9ce72 | No explicit arguments shown | Before the decrement, when the initial offset-8 test is nonzero |
| F-0923d65e0f4b | No explicit argument on the nonzero-result branch; current slot value supplied on the zero-result branch | After the decrement and, where applicable, after the callback |
| F-a600efb7b035 | No explicit arguments shown | Before the second offset-8 read on one terminal-selection path |
| F-feee9156a257 | Retained object address or freshly read offset-8 value | Annotated non-returning terminal operation |
| Indirect callback | Resolved through the offset-0 dispatch chain; no explicit arguments shown | Optional operation on the zero-result branch, before F-0923d65e0f4b |

Dependency implementations, effective calling conventions, implicit inputs, global effects, synchronization, ownership changes, and failure modes are **documented_only** unknowns. No dependency return value is visibly consumed by this function.

## Acceptance examples

These examples are conditional static expectations, not executed tests. Assume valid, nonoverlapping slot, object, and dispatch storage; representable arithmetic; no concurrent mutation; and normally returning nonterminal dependencies. Any required state preservation across an external call is stated explicitly rather than implied by normal return.

1. **Initially empty slot.**  
   A valid slot contains zero. The function returns its address without a dependency call, object access, or slot write.

2. **Count remains nonzero.**  
   The slot names an object whose offset-8 field is zero and whose count is 3. The preliminary operation is skipped. The function writes 2 to the count, then calls F-0923d65e0f4b with no explicit argument shown. If that call returns with the slot still writable, the function clears the slot and returns its address. The count write precedes any changes the dependency may make.

3. **Preliminary operation redirects the decrement.**  
   The initial object’s offset-8 field is nonzero. F-338d16f9ce72 replaces the slot with another valid object address and returns. The decrement accesses the replacement object, not an entry-address snapshot. Its computed result determines the branch.

4. **Zero input count is not rejected.**  
   The preliminary operation is skipped, and the count read yields zero. The function writes −1 and selects the nonzero-result branch. It does not interpret the original zero as a final-reference condition.

5. **Zero result with callback skipped.**  
   The initial object has offset 8 equal to zero, so F-338d16f9ce72 is skipped. The count read yields 1, and the subsequent offset-0 eligibility read yields zero. The function writes a count of zero, skips the callback, and supplies the current slot value to F-0923d65e0f4b. If that dependency clears the slot and returns, the function performs its final zero store and returns.

6. **Callback changes the explicit dependency argument.**  
   The initial offset-8 field is zero, so the preliminary operation is skipped. The count read yields 1, and the later callback eligibility read and dispatch resolution identify a valid callback. If that callback replaces the slot and returns, F-0923d65e0f4b receives the replacement value. If the callback clears it, the supplied value is zero.

7. **Preliminary mutation prevents an expected zero-result branch.**  
   The entry object has count 1 and a nonzero offset-8 field. F-338d16f9ce72 changes the count to 2 while leaving the slot directed at that object. The decrement then produces 1. The function takes the nonzero-result branch and calls F-0923d65e0f4b with no explicit argument shown; entry count 1 did not guarantee callback processing.

8. **Preliminary mutation prevents a callback.**  
   The preliminary operation runs, leaves the count read at 1, but clears the offset-0 field before returning. Assuming no intervening change, the decrement produces zero and the callback is skipped. An entry nonzero dispatch field did not guarantee invocation.

9. **Direct terminal selection.**  
   Following the zero-result F-0923d65e0f4b call, the slot names a valid object whose offset-8 field is zero. F-feee9156a257 receives that object address. Under its non-return annotation, no final slot clear or ordinary return follows.

10. **Fresh terminal argument from the retained object.**  
    The post-call slot names an object with nonzero offset 8. F-a600efb7b035 changes that field and replaces the slot with another address, while keeping the retained object readable. F-feee9156a257 receives the changed field from the retained object, not a field from the replacement object. The changed field may be zero.

## Uncertainties

- **documented_only — Ownership:** Reference release, destruction, and storage reclamation are plausible interpretations, not proven contracts.
- **documented_only — Dependency behavior:** State preservation, implicit arguments, global accesses, lifetime changes, and failure behavior are unavailable.
- **documented_only — Invocation discrepancy:** The two visible argument forms for F-0923d65e0f4b do not establish its actual prototype.
- **documented_only — Terminal behavior:** Non-return annotations are present, but the implementations and behavior if they unexpectedly return are unresolved.
- **documented_only — Arithmetic boundaries:** Signed decrement overflow and address-arithmetic boundary behavior are not established.
- **documented_only — ABI and layout:** Four-byte accesses are visible; complete types, alignment, and broader platform rules are not.
- **documented_only — Aliasing and lifetime:** No general nonaliasing or cross-call lifetime guarantee is supplied.
- **documented_only — Synchronization:** No atomicity, locking discipline, or concurrency guarantee is proven.
- **documented_only — Abnormal exits:** Fault handling, exceptions, and nonlocal control transfers have no established recovery contract.

## Evidence

This contract derives from the supplied static representation and opaque dependency mapping. Prior prose and reviewer feedback are not independent evidence of function behavior.

Established observations include the field offsets, represented widths, decrement, computed-result branch, repeated slot reads, dispatch accesses, visible argument forms, retained post-call object address, repeated offset-8 read, final store, return expression, and terminal-call annotations.

Reference management, finalization, synchronization, and disposal remain inferred intent. Dependency preservation and ownership effects remain unresolved rather than assumed.

No runtime tests have been performed. The acceptance examples describe conditional consequences of the static evidence, not observed executions.
