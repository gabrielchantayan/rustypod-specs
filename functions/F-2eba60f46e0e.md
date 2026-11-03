<!-- Trusted provenance: F-2eba60f46e0e; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-2eba60f46e0e

## Purpose

F-2eba60f46e0e conditionally obtains an object from shared state and invokes an operation on that object using a caller-supplied 32-bit value.

The established behavior is:

- A zero first argument causes a normal return of zero without object lookup or dependency calls.
- A nonzero first argument permits lazy initialization when a shared object slot is zero.
- Initialization requests an operation with the value 24, obtains a candidate object from a second dependency, writes selected object fields, and publishes the object.
- The selected object receives an indirect operation with the address of a local 32-bit copy of the first argument.
- Normal completion after that indirect operation returns one, without examining the operation’s result.

Lazy construction and virtual dispatch are inferred intent. The writes, reads, call ordering, and return values described below are established by the supplied static material. In particular, the apparent allocation-failure check occurs **after** writes through the candidate pointer; it does not establish safe handling of an ordinary null allocation result.

## Interface

### Arguments

The recovered interface has four arguments:

| Position | Width and interpretation | Established use |
|---|---|---|
| First | 32-bit value, represented as a signed integer | Tested for equality to zero; copied into a local 32-bit slot for the indirect operation |
| Second | Opaque 32-bit value | No explicit use |
| Third | Opaque 32-bit value | No explicit use |
| Fourth | 32-bit value, represented as a signed integer | Copied into the local slot on the initialization path, before either direct dependency call |

The first argument is not directly dereferenced. Every nonzero bit pattern, including a value interpreted as negative, passes the initial gate. The supplied function establishes no range restriction beyond the zero/nonzero distinction.

The second and third arguments do not contribute to any explicit read, write, branch, or call argument in this function.

The fourth argument has a local-storage effect on the initialization path. It is not explicitly supplied to either direct dependency. Before the final indirect operation, the same local slot is overwritten with the first argument. Its broader significance cannot be established without the dependency interfaces and calling convention.

### Result

The result is a 32-bit word:

- Zero on the initial zero-argument path.
- Zero on the explicit candidate-null branch, if execution reaches that branch after the preceding candidate writes and publication.
- One after the indirect operation returns normally.

There is no established result for a memory fault, a dependency that does not return, or another nonlocal transfer of control. One is not a verified indication that the indirect operation accepted, retained, or successfully processed the value.

### Relevant storage model

The recovered accesses use 32-bit words for pointers and ordinary integer fields. Object and table offsets are byte offsets. No byte order is established by the supplied material.

The function accesses:

- A global location containing a state-base pointer.
- A 32-bit object-pointer slot at state-base offset 4.
- A separate global location containing the value installed in the candidate’s first word.
- A local 32-bit slot whose address is passed to the final indirect operation.

The state-base pointer must support the accesses actually taken. The function contains no explicit check that this pointer is nonzero or otherwise valid.

## Behavior

### 1. Snapshot shared state and apply the initial gate

The function first reads the global state-base value and retains it locally.

It then tests the first argument. If that argument is zero, it returns zero immediately. This path still includes the initial global state-base read, but it does not dereference the state base, inspect the object slot, initialize an object, or invoke either direct dependency.

For a nonzero first argument, the initialization gate reads the object slot at offset 4 from the state base obtained through the global location.

The supplied representation distinguishes the retained state base from the global expression used for this gate. Later publication and final lookup use the retained base. No synchronization or guarantee that concurrent changes cannot occur is present. Ordinary unchanged global state makes these refer to the same state object; behavior under concurrent replacement is not established.

### 2. Optionally initialize an object

If the gate’s object-slot value is nonzero, initialization is skipped.

If the gate’s object-slot value is zero, the following sequence occurs:

1. The fourth argument is stored into the local 32-bit slot.
2. F-8157a180d901 is invoked with the explicit argument 24.
3. F-e194f3fbdfaa is invoked with no explicit arguments shown.
4. The result of F-e194f3fbdfaa is used as the candidate object address.
5. The candidate’s word at offset 0 is assigned the value read from the separate global initialization-value location.
6. The candidate’s byte at offset 16 is assigned one.
7. The candidate’s 32-bit word at offset 20 is assigned zero.
8. The candidate address is stored in the object slot at offset 4 of the retained state base.
9. Only then is the candidate compared with zero. A zero comparison result causes a return of zero.

The return value of F-8157a180d901 is not explicitly consumed. The relationship between its argument 24 and the candidate obtained from F-e194f3fbdfaa is unknown. Treating the pair as an allocation sequence is an inference, not an established dependency contract.

The explicit initialization layout is:

| Candidate-relative offset | Write width | Written value |
|---|---:|---|
| 0 | 4 bytes | Value read from a separate global location |
| 16 | 1 byte | One |
| 20 | 4 bytes | Zero |

The highest explicit write ends at offset 23. Thus, a candidate supporting at least the accessed portion of a 24-byte region is needed for ordinary valid execution of these stores. This does not establish the candidate’s actual allocation size.

Bytes 4–15 and 17–19 are not initialized by this function. The byte write at offset 16 does not clear or assign the next three bytes.

### 3. Reload the selected object and invoke its operation

After either successful initialization or skipped initialization, the function reads the object pointer from offset 4 of the retained state base.

It then overwrites the local 32-bit slot with the first argument.

The indirect operation is selected by:

- Reading the first 32-bit word of the selected object as a table address.
- Reading the callable entry at byte offset 28 within that table.
- Calling that entry with the selected object and the address of the local slot.

The table entry is therefore seven 32-bit words beyond the table base. This is an offset description, not evidence about the number or meaning of other table entries.

The function does not check the selected object, table address, or callable entry for validity before use. An object obtained from the preexisting-object path must already have a usable first word and operation entry. The initialization path requires the installed first word to support the same dispatch.

The local slot is writable by the indirect operation because its address is supplied. Any value written there is not subsequently inspected by this function.

### 4. Complete normally

If the indirect operation returns normally, the function returns one.

No explicit test of the indirect operation’s return value occurs. No subsequent object-slot read, cleanup operation, or release is shown.

## Side effects

### Direct memory effects

The function can directly:

- Read the global state-base location.
- Read a shared object-pointer slot.
- Write three candidate fields during initialization.
- Publish the candidate into the retained state base’s object slot.
- Read the selected object’s first word and its table’s operation entry.
- Write the local slot, whose address becomes visible to the indirect operation.

The initial object lookup is a gate, not a reservation. There is no visible lock, atomic compare-and-exchange, or second emptiness check before publication. Concurrent initialization, reentrant behavior, and publication visibility are not defined by this function.

### Dependency effects

Either direct dependency may alter global state, object storage, or other memory according to its unknown contract. No preservation guarantee is established.

In particular, the retained state-base pointer remains the publication and final-lookup base even if a dependency changes the global state-base location. The initialization value installed at candidate offset 0 is read after both direct dependency calls; it is not a value captured before those calls.

The indirect operation may likewise change memory or global state. The function’s return of one is conditional only on that operation returning normally, not on preservation of the object or shared slot.

### Ownership and lifetime

The function performs no explicit release or destruction.

Publication makes the candidate address reachable through the retained state object, but does not establish who owns the candidate, who must eventually release it, or whether it was newly allocated.

The indirect operation receives a pointer to local storage, not caller-owned storage. That storage is valid during the invocation. The supplied material does not establish permission to retain the pointer or use it after F-2eba60f46e0e returns.

### Failure behavior

The candidate-null check follows all candidate-field writes and publication. Under an ordinary memory model where address zero is not writable, a zero candidate causes an invalid access before the explicit zero return can be reached.

Consequently, this function must not be specified as safely returning zero on allocation failure. It also establishes no rollback for a failed or interrupted initialization sequence.

## Dependencies

| Dependency | Established interaction | Unknown contract |
|---|---|---|
| F-8157a180d901 | Called only on the initialization path, with the explicit value 24; its result is not explicitly consumed | Purpose, return type, failure behavior, memory effects, and relationship to the next dependency |
| F-e194f3fbdfaa | Called after F-8157a180d901; no explicit arguments are shown; its result is used as the candidate address | Allocation or retrieval behavior, size, alignment, initialization, ownership, failure signaling, and hidden calling-convention inputs |
| Indirect object operation | Selected from table byte offset 28; receives the selected object and a pointer to a local 32-bit value | Callable signature beyond the shown arguments, return meaning, mutation behavior, lifetime rules, and error behavior |

No additional opaque ID is supplied for the indirect operation. Its identity is therefore described by its selection mechanism, not guessed.

## Acceptance examples

These are conditional static acceptance examples, not executed tests.

### Zero first argument

Given a zero first argument:

- The global state-base value is read.
- The function returns zero.
- No state-base dereference, initialization call, or indirect object operation is performed.

This expectation does not require the state-base value itself to point to valid storage, because this path does not dereference it.

### Existing valid object

Given a nonzero first argument, stable state, and a valid nonzero object slot with a usable dispatch table:

- Neither direct dependency is called.
- The indirect operation receives that object and a pointer to a local 32-bit copy of the first argument.
- A normal return from the operation is followed by a result of one.

The fourth argument has no explicit effect on this path.

### Empty slot and valid candidate

Given a nonzero first argument, an empty object slot, normally returning dependencies, and a valid candidate:

- F-8157a180d901 receives 24 before F-e194f3fbdfaa is called.
- The three candidate fields are written in the documented order.
- Publication occurs after those writes.
- The selected object is reloaded through the retained state base.
- The indirect operation receives the first argument’s value, not the fourth argument’s earlier local value.
- Normal completion returns one.

### Candidate contains preexisting bytes

Given a candidate with arbitrary existing contents:

- Only offsets 0–3, 16, and 20–23 are explicitly overwritten during initialization.
- Other bytes cannot be required to become zero or take any particular value solely because of this function.

### Zero candidate

Given that F-e194f3fbdfaa returns zero:

- The next candidate access is a word write at offset 0, not a null check.
- No ordinary safe zero-return outcome is guaranteed.

Only if the preceding writes and publication can complete does the later comparison establish a zero return.

### Indirect operation modifies the local value

Given that the indirect operation changes the pointed-to 32-bit value and returns normally:

- The changed value is not used to determine the result.
- The result remains one.
- No copy-back to caller storage is shown.

## Uncertainties

The following remain **documented_only**:

- Actual dependency signatures and calling convention.
- Whether 24 denotes an allocation size or another request.
- Whether the two direct dependencies cooperate through hidden state.
- The semantic meaning of the fourth argument’s temporary local store.
- Candidate ownership, allocation size, alignment, and release obligations.
- The semantic meaning of the byte at offset 16 and word at offset 20.
- The identity and complete contract of the indirect operation.
- Global mutability, thread safety, reentrancy, and memory-ordering guarantees.
- Behavior of invalid accesses and nonlocal dependency failures in the execution environment.

Allocation, construction, and virtual-method terminology express inferred intent only. They do not override the established access ordering or supply missing safety guarantees.

## Evidence

The evidence consists solely of the supplied static function representation and the supplied mapping to opaque function IDs.

It establishes explicit argument uses, 32-bit word and one-byte accesses, field offsets, branch conditions, call order, publication order, indirect-operation selection, and normal return constants. It does not establish the implementations or contracts of the dependencies.

No implementation access, runtime tests, or dynamic observations were used. The acceptance examples are consequences of the static evidence under their stated preconditions, not reports of observed execution.
