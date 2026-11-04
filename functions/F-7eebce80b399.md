<!-- Trusted provenance: F-7eebce80b399; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-7eebce80b399

## Purpose

F-7eebce80b399 conditionally updates a state object, invokes dependencies, and returns a 64-bit value containing the original state-pointer representation and a branch-dependent 32-bit value.

An initial 32-bit read at state byte offset 16 controls the branch. If that sample is zero, the function makes no dependency calls and performs no explicit externally addressed writes. The high result half comes from the second input.

If the initial sample is nonzero, the function:

- Reads state offset 16 **again**, using this second sample to locate an auxiliary field.
- Passes three addressable local slots and the state pointer to F-2ad0bc7d0efc.
- Reads state offset 16 a third time, after that dependency returns, to select an object for linking.
- Repeatedly removes the object currently referenced by state offset 0 and invokes F-ff1bdc376dcd twice for each removed object.
- Obtains the high result half from a local slot exposed to the first dependency.

The three offset-16 reads have distinct roles. Their values are equal only if the field remains unchanged across the relevant reads. “Linking,” “chain,” and “removal” describe the visible field operations; they do not establish ownership or allocation semantics.

## Interface

### Inputs

| Input | Width and interpretation | Established use |
|---|---|---|
| State pointer | Pointer represented in a 32-bit address domain | Addresses state fields and supplies result bits 0–31. |
| Secondary value | 32 bits, presented as signed | Supplies result bits 32–63 on the initial-zero branch. |
| Third value | 32 bits, otherwise uninterpreted | Not used by the visible body. |
| Fourth value | 32 bits, otherwise uninterpreted | Initializes a local slot passed by address to F-2ad0bc7d0efc. |

The secondary value is not used in a signed comparison or arithmetic operation. Its exact 32-bit representation is retained when selected for the result.

### State and referenced layouts

All offsets below are byte offsets. Every listed field access transfers 32 bits.

| State offset | Established role |
|---|---|
| 0 | Current object address for removal; zero terminates the loop. |
| 4 | Existing link value; replaced during linking. |
| 16 | Sampled first for the branch, again before the first dependency for auxiliary addressing, and again afterward for link-destination selection. |

No behavior is established for other state fields.

The object addressed by the **second pre-dependency sample** of state offset 16 must provide a readable field at offset 8. This address is not necessarily the initially sampled nonzero value.

The object addressed by the **post-dependency sample** of state offset 16 must provide a writable field at offset 12. It need not be the object used for the auxiliary read.

Objects reached through state offset 0 expose:

| Object offset | Established role |
|---|---|
| 0 | Successor value copied into state offset 0. |
| 4 | Second argument of the first F-ff1bdc376dcd call for that object. |
| 8 | First argument of that call. |

These are partial access layouts, not complete object definitions or minimum allocation sizes.

### Result

The normal result is a 64-bit concatenation:

- Bits 0–31: original state-pointer representation.
- Bits 32–63: secondary input on the initial-zero branch; otherwise, the local result-slot value sampled at the terminating loop condition.

The high half is not sign-extended into the low half, and the halves are not arithmetically added. Caller interpretation of the result is unknown.

## Behavior

### Initial sample and branch

The function first reads state offset 16 into an addressable 32-bit local slot, called the **result slot** in this contract.

This initial sample alone controls entry into the nonzero branch.

If the sample is zero, the function:

- Does not perform a second read of state offset 16.
- Does not access an auxiliary object.
- Does not invoke either dependency.
- Does not perform the linking or removal writes.
- Returns the secondary input as the high result half.

A nonzero state offset-0 field does not independently trigger processing.

### Second selector read and dependency preparation

If the initial sample is nonzero, the function reads state offset 16 a second time, before calling any dependency. This second sample selects the address used for the auxiliary offset-8 read.

The function then reads the 32-bit field at that address plus 8 into a separate **auxiliary slot**. It next initializes a **fourth-input slot** from the fourth input.

The result slot still contains the first sample. The second sample selects the auxiliary address; it does not replace the result-slot value.

There is no zero check or validity check on the second sample. A nonzero first sample followed by a zero second sample does not select the initial-zero return path. The function proceeds with the auxiliary read using the second sample and the offset of 8.

F-2ad0bc7d0efc is then called with these arguments, in positional order:

1. Address of the fourth-input slot.
2. Original state pointer.
3. Address of the auxiliary slot.
4. Address of the result slot.

At call entry:

- The fourth-input slot contains the fourth input.
- The auxiliary slot contains the field read using the second selector sample.
- The result slot contains the initial selector sample.

The dependency’s direct return value is not consumed. The function does not restore state or local-slot contents after the dependency returns. The fourth-input and auxiliary slots are not explicitly read again by this function; that does not establish whether their addresses escape or whether the dependency changes them.

### Post-dependency selection and linking

After F-2ad0bc7d0efc returns, the function reads state offset 16 again. This third sample selects the link destination, independently of both earlier samples and the result slot.

The visible linking sequence is:

1. Obtain the current state offset-4 value.
2. Write that 32-bit value into the selected object’s offset-12 field.
3. Write the selected object’s address into state offset 4.

The selected address is retained for these linking operations. The destination-field write precedes the state offset-4 write.

There is no zero check on the post-dependency selector. If it is zero or otherwise unusable, the function does not take a clean early-return path; it proceeds toward the offset-12 write.

The function does not explicitly clear or replace state offset 16. This statement concerns the addressed field operations: an overlapping layout could make a write through another address affect it, and permitted alias relationships are not established.

### Removal loop

Following linking, each loop-condition evaluation:

1. Reads state offset 0 into the current-object pointer.
2. Samples the result slot.
3. Continues only if the current-object pointer is nonzero.

For each nonzero current-object pointer, the function:

1. Reads the object’s offset-0 successor.
2. Stores that successor into state offset 0.
3. Reads the object’s offset-8 and offset-4 fields to prepare the first dependency call.
4. Calls F-ff1bdc376dcd with those values, in that positional order, followed by a 32-bit zero.
5. After that call returns, calls F-ff1bdc376dcd with the saved current-object address, a 32-bit one, and a 32-bit zero.
6. Reevaluates the loop condition.

The relative order of the offset-8 and offset-4 argument-field reads is not established. Both are prepared after the state-head update and before the first call.

The second call uses the saved object address. It does not reload that address from state offset 0 or reread the object’s fields. No wrapper-level validity check follows the first call.

The next iteration rereads state offset 0 rather than following a privately retained successor. Consequently, dependency-induced state-head changes can redirect processing or terminate it.

There is no visible cycle detection, iteration bound, or visited-object tracking.

### Final result selection

On the nonzero branch, the high result half is the result-slot sample taken during the loop-condition evaluation that observes a zero current-object pointer.

This sample occurs even when the first condition evaluation finds the chain empty, so no removal calls are required to select the high half.

The high half is not necessarily any of the following:

- The secondary input.
- The second selector sample used for auxiliary addressing.
- The post-dependency selector used for linking.
- The result-slot value immediately after the first dependency returns, if later effects change that slot.

If the result slot remains unchanged throughout the nonzero branch, its final sample equals the initial selector sample.

## Side effects

### Explicit writes and calls

On the initial-zero branch, there are no explicit externally addressed writes and no dependency calls.

On the nonzero branch, the function explicitly performs:

- One write to the post-dependency selected object’s offset-12 field.
- One write to state offset 4.
- One write to state offset 0 for each processed object.
- One call to F-2ad0bc7d0efc.
- Two ordered calls to F-ff1bdc376dcd for each processed object.

Aliasing can make these addressed writes affect overlapping storage.

The function also initializes and uses local slots. Three local addresses are exposed to F-2ad0bc7d0efc. Their exposure does not establish valid storage lifetime beyond this invocation.

No direct allocation or release primitive is identified in the supplied body. The removal-call pattern suggests cleanup as an interpretation, but does not prove that any object or payload is released. No assumption is made that either dependency preserves globals, retains allocations, releases memory, or refrains from retaining pointers.

### Preconditions and failure boundaries

Normal completion requires all executed accesses to be valid and all invoked dependencies to return normally.

In particular:

- State offset 16 must be readable for the initial sample.
- If that sample is nonzero, state offset 16 must be readable again for auxiliary selection.
- The address selected by that **second** sample must support the offset-8 read. Initial nonzeroness alone does not establish this.
- After F-2ad0bc7d0efc returns, the required state fields must remain accessible.
- The post-dependency selected address must support the offset-12 write.
- State offset 4 and, during removal, offset 0 must support the specified accesses.
- Each processed object must support its executed successor and argument-field reads.
- The loop must eventually observe a zero state-head value for normal termination.

There are no explicit null-pointer, address-validity, alignment, aliasing, or chain-integrity checks. No error result or rollback path is visible. If execution fails after a write, no restoration of earlier state is specified.

The visible arithmetic is fixed-offset addressing. There is no length calculation, multiplication, allocation-size computation, or signed arithmetic on the secondary input. Address overflow, alignment requirements, byte order, and fault behavior remain platform-dependent unknowns.

## Dependencies

### F-2ad0bc7d0efc

Called once only when the first state offset-16 sample is nonzero, provided execution reaches the call.

Its four arguments are the fourth-input-slot address, state pointer, auxiliary-slot address, and result-slot address. The auxiliary slot derives from the second selector read; the result slot initially derives from the first.

Its return value is ignored. Later behavior consumes freshly read state fields and samples the exposed result slot. Its mutation, pointer-retention, ownership, global-state, and failure contracts are undocumented here.

### F-ff1bdc376dcd

Called twice for each fully processed object:

| Call | First argument | Second argument | Third argument |
|---|---|---|---|
| First | Object offset-8 value | Object offset-4 value | 0 |
| Second | Saved object address | 1 | 0 |

Arguments are represented as 32 bits. Direct return values are ignored.

The constants do not establish sizes, flags, allocator classes, or release instructions. Normal return from the first call is required to reach the second. Both calls can affect later behavior through effects not constrained by the supplied evidence.

## Acceptance examples

These are conditional static acceptance cases, not runtime observations. Each assumes the described accesses are valid and dependencies return normally unless stated otherwise.

1. **Zero initial selector with a nonempty head.**  
   Given an initial selector sample of zero, a nonzero state head, and secondary input `0xFFFFFFFF`, no dependency is called and no explicit external write occurs. There is no second selector read. The result high half is `0xFFFFFFFF`; the low half is the original state-pointer representation.

2. **Stable selector before the first dependency.**  
   If the first and second selector reads both yield address A, the result slot initially contains A and the auxiliary slot receives A’s offset-8 field. This equality follows from the stated stability condition, not from an unconditional single-sample rule.

3. **Different pre-dependency samples.**  
   If permitted execution conditions produce nonzero A on the first selector read and B on the second, branch entry follows A, the result slot initially contains A, and the auxiliary read uses B plus 8. The first dependency receives the auxiliary value from B, not necessarily from A. Whether concurrent modification can validly produce this case is undocumented.

4. **Second selector becomes zero.**  
   If the first sample is nonzero and the second is zero, the function still attempts the auxiliary read at address zero plus 8. It does not return the secondary input merely because the second sample is zero. Whether that access succeeds is not established.

5. **Three distinct selector values.**  
   Suppose the first sample is A, the second is B, and the post-dependency sample is C. The result slot starts with A; the auxiliary read targets B’s offset 8; linking targets C’s offset 12 and then stores C into state offset 4. The final high result half depends on the terminating result-slot sample, not automatically on B or C.

6. **Empty head after linking.**  
   If linking completes and state offset 0 is zero at the first condition evaluation, no removal calls occur. The result high half is the result-slot value sampled during that evaluation.

7. **One object without dependency redirection.**  
   If the post-link head identifies one object with a zero successor, and dependencies do not alter the head, state offset 0 becomes zero before either removal call. The calls receive the specified field values and saved address. The next condition evaluation terminates the loop.

8. **Dependency redirects the head or changes the result slot.**  
   If a removal dependency changes state offset 0, the next condition follows that current value. If an effective write changes the exposed result slot to `0x89ABCDEF` before the terminating sample and no later write changes it, the result high half is `0x89ABCDEF`. This does not establish that a dependency can safely retain a local address beyond its valid lifetime.

## Uncertainties

The following remain **documented_only**:

- Dependency implementations, preservation guarantees, ownership effects, and pointer-retention rules.
- Complete object layouts and semantic field names.
- Whether the structures are intended as lists or the calls as cleanup.
- Caller interpretation of the 64-bit result.
- Valid aliasing relationships and overlapping-storage behavior.
- Concurrency, reentrancy, and memory-access guarantees, including whether changes between selector reads can occur in a valid execution.
- Required alignment, byte order, address-overflow semantics, and fault handling.
- Termination guarantees for cycles or dependency-directed head changes.
- Lifetime of objects and resources after either dependency call.

The three selector reads are established static operations. Their equality, synchronization, and runtime observability under a particular execution model are not established.

## Evidence

The supplied function body and opaque dependency mapping are the sole evidence.

Established static observations include the four inputs, 32-bit field transfers, initial branch sample, separate second pre-dependency selector read, auxiliary-field access, three exposed local addresses, post-dependency selector reread, ordered linking writes, unlink-before-call sequence, repeated head and result-slot sampling, and 64-bit concatenated return.

Structural and cleanup terminology is inferred intent only. No dependency bodies, caller contracts, ownership documentation, or runtime observations were supplied. No runtime tests have been performed.
