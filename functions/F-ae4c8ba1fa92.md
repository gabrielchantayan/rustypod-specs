<!-- Trusted provenance: F-ae4c8ba1fa92; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-ae4c8ba1fa92

## Purpose

F-ae4c8ba1fa92 copies the current contents of a 32-bit global into caller-supplied storage, obtains an address from a dependency, and conditionally performs record linking and list draining. Its 64-bit return value combines a retained, derived address with the final contents of a mutable 32-bit local slot.

The established contract is structural. The supplied source does not establish the application-level meanings of the global, object, records, list entries, or return components. Linking and calls made after unlinking do not establish ownership transfer or memory release.

This specification is based on static evidence only. No runtime tests have been performed.

## Interface

The function has four arguments:

| Argument | Width and interpretation | Established use |
|---|---|---|
| Object | Address of storage accessed in 32-bit units | Receives a 32-bit write at offset zero. Object +32 bytes is passed to F-fd7c65715716. |
| Initial upper component | 32-bit integer | Initializes the local slot used for the upper half of the return value. The conditional path replaces this slot and exposes its address to F-3e95f52e82a5. |
| Initial auxiliary value | 32-bit uninterpreted value | Initializes a local slot that the conditional path replaces with a record field before exposing its address to F-3e95f52e82a5. |
| Initial mutable value | 32-bit uninterpreted value | Initializes a local slot whose address is passed to F-3e95f52e82a5 on the conditional path. |

The function also reads one **32-bit initialization global**. Its current contents, not an established fixed constant, supply the initial object write. Its numerical value, mutability, and semantic role are unknown.

### Return value

The result is a 64-bit concatenation:

- **Upper 32 bits:** final contents of the local upper-component slot.
- **Lower 32 bits:** the address returned by F-fd7c65715716, adjusted downward by 32 bytes.

The upper component is included by its 32-bit representation. A signed interpretation does not sign-extend it across the lower component. The result is not an arithmetic sum of the two components.

### Address notation and accessed layouts

**Anchor** denotes the 32-bit address-bearing result from F-fd7c65715716. **Base** denotes Anchor minus 32 bytes.

The following accesses are established relative to Base:

| Offset | Width | Role established by access |
|---|---:|---|
| +4 | 32 bits | List-head address |
| +8 | 32 bits | Link value copied into a selected record, then replaced with that record’s address |
| +20 | 32 bits | Record-address field controlling the conditional path and reread within it |

A record selected through Base +20 is accessed at:

| Offset | Width | Use |
|---|---:|---|
| +8 | 32 bits | Read into the auxiliary local slot before F-3e95f52e82a5 |
| +12 | 32 bits | Written with the value read from Base +8 after that dependency returns |

A list entry is accessed at:

| Offset | Width | Use |
|---|---:|---|
| +0 | 32 bits | Next-entry address |
| +4 | 32 bits | Second argument of the first F-ff1bdc376dcd call for the entry |
| +8 | 32 bits | First argument of that call |

These are access layouts, not complete type definitions or allocation sizes. Object, Base-relative storage, records, entries, and the initialization global are not proven disjoint.

### Preconditions and arithmetic limits

For normal completion, all storage actually read or written must support the corresponding access. Dependency arguments must also satisfy the dependencies’ unknown contracts.

In particular:

- The initialization global must be readable as 32 bits.
- Object must support the initial 32-bit write.
- Base-relative fields must be accessible when their paths are reached.
- Addresses used for record-field and entry-field accesses must designate accessible storage at the required offsets.
- All dependencies actually invoked must return normally for later behavior to occur.

The function performs no explicit address validation.

The supplied representation uses 32-bit address-bearing values and 32-bit fields. Object +32 follows from advancing eight 32-bit units. Base +4, Base +8, and Base +20 correspond respectively to Anchor minus 28, 24, and 12 bytes.

No portable overflow, wrapping, alignment, or invalid-address rule is established. Conditional acceptance examples assume that the required address calculations and accesses are meaningful in the execution environment.

## Behavior

### 1. Copy the initialization global and prepare local state

The function first reads the current 32-bit contents of the initialization global and writes that value to Object +0.

This global read and object write precede initialization of the three local slots and precede the call to F-fd7c65715716. The operation is a copy of a global value at this point in execution; it is not evidence that the value is constant.

The function then initializes:

- The upper-component slot from the second argument.
- The auxiliary slot from the third argument.
- The mutable-value slot from the fourth argument.

It calls F-fd7c65715716 with Object +32 bytes. The returned address becomes Anchor, from which Base and the address of the list-head field are derived.

The dependency may observe or alter storage affected by the initial write. No preservation guarantee applies to Object, list storage, record storage, or globals across this call.

### 2. Decide whether to enter the conditional path

After F-fd7c65715716 returns, the function reads the record-address field at Base +20.

If that read yields zero, the conditional path is skipped:

- F-3e95f52e82a5 is not called.
- F-ff1bdc376dcd is not called.
- No explicit record-linking stores occur.
- After this decision, the function performs no explicit list-head access or draining.
- The return upper component is the initial upper-component argument.
- The return lower component is Base.

This does not establish that the list was preserved throughout the invocation. The earlier Object +0 write may alias list storage, and F-fd7c65715716 may mutate it before the zero-record decision.

A nonzero list head alone does not trigger draining. The record-address decision gates both linking and draining.

### 3. Prepare the addressed arguments

If the decision read is nonzero, the function performs further reads of Base +20:

1. It rereads the record address, reads the 32-bit field at that address +8, and replaces the auxiliary slot with that field value.
2. It rereads the record address again and replaces the upper-component slot with that address.

The decision read and these two preparation reads are distinct. They are not an established atomic snapshot. If the record-address field changes between reads, the copied field value and the copied address may come from different states.

There is no further nonzero check before the record +8 read.

The function calls F-3e95f52e82a5 with four addresses, in this positional order:

1. Address of the mutable-value slot initialized from the fourth argument.
2. Address of the list-head field at Base +4.
3. Address of the auxiliary slot now holding the copied record +8 value.
4. Address of the upper-component slot now holding the copied record address.

The dependency’s return value is unused. The supplied addresses permit it to modify the corresponding storage; its actual effects remain unknown.

After the call, the function does not explicitly read the mutable-value or auxiliary slots again. It does read the upper-component slot when constructing its final return value.

### 4. Link the post-call selected record

After F-3e95f52e82a5 returns normally, the function rereads Base +20 and retains that record address for the following stores.

It then:

1. Reads the 32-bit value at Base +8.
2. Writes that value to the selected record’s +12 field.
3. Writes the selected record’s address to Base +8.

The record selected here need not be the record used during preparation. It also need not equal the final contents of the upper-component slot.

There is no nonzero check on the post-call record address. If Base +20 has become zero, the function still attempts a 32-bit store at the address obtained by adding 12 bytes to zero. The source does not establish whether such an access succeeds, faults, or has another environment-specific consequence.

The record +12 store precedes the Base +8 store. Aliasing can therefore affect the resulting contents. No non-aliasing rule is enforced.

### 5. Drain the list

After linking, the function repeatedly reads the list head at Base +4.

A zero head ends the loop. For each nonzero head, the function retains that entry address and performs these actions:

1. Reads the entry’s next address from offset +0.
2. Writes that next address into Base +4.
3. Calls F-ff1bdc376dcd with the entry’s 32-bit +8 field, the entry’s 32-bit +4 field, and zero, in that positional order.
4. After the first call returns normally, calls F-ff1bdc376dcd with the retained entry address, the 32-bit value one, and zero.

The head is updated before either call associated with the entry. The +8 and +4 argument fields are read after that update; their relative read order is not established. If the head field aliases either argument field, the head update can affect the values supplied to the first call.

The second call uses the retained entry address, not a newly loaded head. It does not require another explicit read of the entry’s fields. This does not establish that the first call preserves entry storage or that either call releases it.

After both calls return, the function rereads Base +4. It does not privately traverse a saved next-address chain. Dependency mutations of the head can therefore redirect or terminate subsequent processing.

There is no cycle detection, iteration limit, or independent progress check. A finite initial chain does not guarantee termination if dependencies rewrite the head.

### 6. Assemble the result

After skipping the conditional path or reaching a zero list head, the function returns the final upper-component slot concatenated with Base.

Base remains derived from the original F-fd7c65715716 result. It is not recomputed from later record, list, or global contents.

## Side effects

Direct nonlocal accesses include:

- Reading the 32-bit initialization global before the initial object write.
- Writing the value read from that global to Object +0.
- Reading the Base-relative record-address field as described above.
- On the conditional path, reading a record’s +8 field.
- Writing the post-call selected record’s +12 field.
- Replacing Base +8 with that selected record’s address.
- Reading the list head and entry fields during draining.
- Replacing Base +4 once per processed entry.

The function itself has no explicit store to the initialization global. This is not a guarantee that the global remains unchanged: aliasing or dependency effects may change it.

F-3e95f52e82a5 receives addresses of three local slots and the list-head field. All dependencies may have additional effects not visible in the supplied body. Ignoring a dependency’s return value does not imply that it preserves memory or globals.

No explicit allocation operation appears. The paired F-ff1bdc376dcd calls do not establish deallocation, allocator pairing, reference counting, or ownership transfer.

There is no explicit error result, status check, rollback, or restoration path. Earlier writes and calls have already occurred if a later access or dependency fails. The behavior following an abnormal dependency exit is not established.

Repeated reads do not establish synchronization or thread safety.

## Dependencies

| Opaque ID | Invocation visible here | Unresolved contract |
|---|---|---|
| F-fd7c65715716 | Called once with Object +32; supplies the 32-bit address used as Anchor | Input requirements, result validity, memory and global effects, failure behavior |
| F-3e95f52e82a5 | Called once only on the nonzero-record path, with the four addressed arguments described above; return value unused | Mutations of slots, list, records, or globals; pointer retention; ownership; failure behavior |
| F-ff1bdc376dcd | Called twice per processed entry, first with two entry fields and zero, then with the entry address, one, and zero; return values unused | Argument meanings, resource effects, memory and global mutations, failure behavior |

Passing local-slot addresses does not establish that retaining those addresses after return is permitted. No lifetime or retention contract for the dependencies is supplied.

## Acceptance examples

These examples are conditional consequences of static evidence, not executed tests.

### Current global value supplies the initial write

Suppose the initialization global reads as a particular 32-bit value and Object supports the write.

- That value is written to Object +0 before F-fd7c65715716 is called.
- No particular numerical value is required by the supplied source.
- If the global has a different value on another invocation, the copied value can differ.
- Object’s final contents remain dependent on aliasing and later dependency effects.

### Zero record decision with a nonempty list

Suppose the decision read at Base +20 yields zero, while the list head is nonzero.

- F-fd7c65715716 has already been called.
- Neither conditional-path dependency is called.
- After the decision, this function performs no explicit list-head access or draining.
- The upper return component equals the second input argument.
- List preservation across the entire invocation is not guaranteed: the initial object write may alias list storage, and the first dependency may have changed it.

### Stable record and empty list

Suppose the record address remains nonzero and stable, the relevant storage is valid and nonoverlapping, and the head is zero after F-3e95f52e82a5.

- The record’s +8 value is available through the third addressed argument.
- The record’s +12 field receives the Base +8 value read after that dependency.
- Base +8 becomes the record address.
- No F-ff1bdc376dcd call occurs.
- The upper return component reflects the final upper-component slot, including any dependency modification.

### Dependency replaces the selected record

Suppose F-3e95f52e82a5 changes Base +20 from one valid record address to another.

- Preparation may already have copied the original record’s +8 value.
- Linking uses the replacement record.
- The upper return component follows its local slot; it does not automatically follow the replacement record.

### Two-entry list without relevant mutation

Suppose a valid two-entry acyclic list remains after linking, relevant storage does not overlap, and the dependencies do not alter the head or entry fields used by traversal.

- The first entry is unlinked before its two calls.
- Its field-based call precedes its address-based call.
- The second entry is processed in the same order.
- Four F-ff1bdc376dcd calls occur.
- The head is zero when the loop exits normally.

### Dependency rewrites the head

Suppose a call for a removed entry changes Base +4, and the later call for that entry does not replace that change.

- The next loop decision uses the changed head.
- An original entry may be skipped, or a newly installed entry may be processed.
- Termination does not follow solely from the initial chain being finite.

### Dependency clears the record address

Suppose F-3e95f52e82a5 clears Base +20.

- The function does not retroactively take the zero-record shortcut.
- It attempts the post-call record-field store at zero plus 12 bytes.
- No checked error result or recovery behavior is established.

## Uncertainties

The following remain **documented_only**:

- The initialization global’s numerical value, mutability, and semantic role.
- Complete layouts, alignment requirements, and allocation boundaries.
- Whether Base identifies an allocation boundary or interior storage.
- Permitted aliasing and external mutation.
- Ownership of records, entries, and entry-associated values.
- Whether either F-ff1bdc376dcd call releases resources.
- Dependency internals, pointer-retention rules, and failure mechanisms.
- Application-level meanings of the return components.
- Environment-specific behavior for invalid or overflowing address calculations and accesses.
- Termination for cyclic or dependency-mutated structures.

**Inferred intent:** the post-call stores resemble linking a record into another structure, and the loop resembles work-list cleanup. These interpretations do not establish resource semantics or additional required behavior.

## Evidence

Established static evidence consists of the supplied function body’s widths, global load, stores, fixed-offset accesses, branch, dependency calls, loop, and 64-bit concatenation.

The body specifically supports a read of the current 32-bit initialization global followed by a write to Object +0, before local-slot initialization and the first dependency call. It does not support describing the copied value as a fixed constant.

The evidence also supports distinct record-address reads, post-call record selection, ordered linking stores, unlink-before-call processing, head rereads, and return assembly. The zero-record branch supports omission of subsequent explicit list processing, not preservation of list storage across earlier writes or dependency execution.

Dependency internals, ownership, runtime validity, and application-level intent are not established. No runtime execution, tests, or external implementation inspection have been performed.
