<!-- Trusted provenance: F-adc7c0d4f0b6; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-adc7c0d4f0b6

## Purpose

F-adc7c0d4f0b6 chooses between two execution paths:

- An **active path** that prepares temporary state from an indexed record, checks that state, selects either an indexed record or a fixed fallback record, and returns a dependency’s result.
- An **inactive path** that conditionally performs guarded initialization and returns a shared global value.

These are behavioral descriptions, not established domain meanings. The supplied static representation does not identify what the records represent, what the shared value contains, or whether the guarded operation constructs an object, registers a resource, or performs another kind of initialization.

The contract below distinguishes directly visible behavior from interpretation and unresolved dependency behavior. No runtime tests have been performed.

## Interface

### Inputs

The function accepts four inputs:

| Input | Visible representation | Use |
|---|---|---|
| Record base | 32-bit integer-form address | Base for indexed and fixed-offset address calculations on the active path. |
| Forwarded value | Opaque 32-bit value | Passed unchanged to two active-path dependencies. |
| Initial temporary address | 32-bit integer-form value | Initializes an address-like temporary whose storage is exposed to dependencies. |
| Initial temporary word | Opaque 32-bit value | Initializes a second temporary whose storage is exposed to dependencies. |

“Integer-form address” means that the supplied representation performs integer arithmetic on the value and uses resulting values as addresses. It does not establish a public pointer type, address-space qualification, alignment guarantee, or ownership contract.

The two temporaries are distinct writable 32-bit storage locations. Their initial values are the third and fourth inputs, respectively. Although the supplied representation places them in local storage, it does not establish an externally meaningful aggregate layout or permit treating them as an array.

### Output

The function returns one opaque 32-bit value:

- On the active path, the return value is exactly the value returned by F-b4a9589b2ea9.
- On the inactive path, the return value is the shared result value read at the final return point.

No visible output convention identifies particular values as success, failure, null, a handle, or a pointer.

### Record layout used by this function

Only the following layout facts are established:

| Location | Visible use |
|---|---|
| Record base + `0x54` | A 32-bit index-like field, represented as a signed integer. |
| Record base + index × `0x1c` | Address passed as an indexed record. |
| Indexed record + `0x10` | A 32-bit word loaded into the second temporary. |
| Record base + `0x38` | Fixed fallback record address. |

The stride is 28 bytes. The fixed fallback offset is 56 bytes, numerically two strides from the base. That arithmetic does not establish an array length, a reserved-record convention, or the validity of any index.

### Preconditions and arithmetic limits

The active path contains no visible checks for a null base, index bounds, alignment, address overflow, or readable record storage. Valid execution therefore depends on the calculated addresses being usable for the accesses and dependency calls that actually occur.

The index load and address calculations use 32-bit quantities in the supplied representation. An implementation or consumer must not silently reinterpret the index as a wider unsigned quantity. Conversely, the evidence is insufficient to specify portable signed-overflow behavior or to promise modulo-\(2^{32}\) address wrapping.

The inactive path does not visibly dereference the record base. Its validity instead depends on the shared guard storage, global values, and dependency contracts.

## Behavior

### Initial state and path selection

Before calling any dependency, the function initializes:

- The address-like temporary from the third input.
- The second temporary from the fourth input.

It then calls F-8094c3d72052 without explicit arguments.

A nonzero result selects the active path. A zero result selects the inactive path. The function does not visibly repeat this mode query.

The mode dependency can have effects not shown here. The subsequent behavior must be understood against the state that exists after that call, not necessarily the state at entry.

### Active path: preparation

The function first reads the index-like field at base + `0x54` and calculates an indexed record address using the 28-byte stride.

It calls F-81ba24aad07e with these arguments, in order:

1. The address of the address-like temporary.
2. The calculated indexed record address.
3. The unchanged second input.

The dependency’s return value is not used by this function. Passing writable temporary storage permits the dependency to replace the address-like temporary. The supplied body does not establish whether it does so, whether it also changes record storage, or whether it affects shared state.

After that call completes, the function reads the index-like field again. It calculates an indexed record address from this new read, loads the 32-bit word at that record’s offset `0x10`, and assigns that word to the second temporary.

Consequently:

- The fourth input’s initial value is overwritten on every active-path execution that reaches this load.
- The record used for this word load need not be the record previously passed to F-81ba24aad07e.
- Changes to the index field made during the first dependency call can affect this load.

The function does not cache the first index value for reuse.

### Active path: checks and selection

The function calls F-eac694802fc1 with:

1. The address of the address-like temporary.
2. The address of the second temporary.

Both temporaries are therefore exposed through writable addresses. Later use of the address-like temporary observes its value after this call.

The next operation depends on F-eac694802fc1’s result:

- If the result is nonzero, the function selects the fixed fallback record and does not call F-9316cc774c14.
- If the result is zero, the function calls F-9316cc774c14 with the current address-like temporary plus `0x14`.

The latter argument is an address-like value computed by adding 20 bytes. The caller does not visibly load a word from that location before passing it. The supplied evidence does not establish whether F-9316cc774c14 reads, writes, or merely interprets that argument.

The final selection is:

| First check | Second check | Selected record |
|---|---|---|
| Nonzero | Not called | Fixed fallback at base + `0x38` |
| Zero | Nonzero | Fixed fallback at base + `0x38` |
| Zero | Zero | Indexed record using another read of the index field |

For the indexed outcome, the function reads base + `0x54` a third time, after both checks have completed, and computes the selected record using that value.

For either fallback outcome, it calculates base + `0x38` without a third index read.

This ordering matters if a dependency changes the index field. The record initially prepared, the record supplying the second temporary, and the final indexed record can all differ.

### Active path: result production

After selecting a record address, the function calls F-b4a9589b2ea9 with:

1. The selected record address.
2. The unchanged second input.

It immediately returns that dependency’s 32-bit result. No additional visible validation, conversion, cleanup, or shared-result substitution occurs.

### Inactive path: guard decision

The function examines bit zero of the shared guard storage.

- If that bit is set, it skips the guarded sequence.
- If that bit is clear, it calls F-b255b26284e6 with the guard reference.
- A zero result from F-b255b26284e6 also skips the guarded sequence.
- A nonzero result enters the guarded sequence.

The direct guard inspection establishes only a low-bit test. The supplied representation does not reliably establish the full width of the guard object, its complete layout, or its synchronization semantics.

A zero guard-acquisition result is not returned as an error. It causes the function to proceed to the shared-result return.

### Inactive path: guarded sequence

When the guarded sequence is entered, these calls occur in order:

1. F-9b9c1c7483f4 receives:
   - The current shared result value.
   - The address of a shared descriptor.
   - The address of the second temporary, which initially contains the fourth input on this path.
2. F-804adc29bbfb receives:
   - The return value from F-9b9c1c7483f4.
   - A first shared auxiliary value.
   - A second shared auxiliary value.
3. F-d08d3ca17b81 receives the guard reference.

The descriptor argument is an address, not a load of the descriptor’s contents.

The auxiliary values are supplied to the second call after the first call has completed. Their prior values must not be assumed unchanged across the first call. Likewise, the guard reference used for the final call must not be assumed immune to earlier dependency effects unless an external contract guarantees that.

The return values of F-804adc29bbfb and F-d08d3ca17b81, if any, do not influence a visible branch or the function’s returned value.

### Inactive path: return

Whether the guarded sequence ran or was skipped, the function returns the shared result value as read at the final return point.

That value is not necessarily the value supplied to F-9b9c1c7483f4. Any intervening dependency may have changed shared storage.

There is no visible direct assignment to the shared result in this function.

## Side effects

### Directly established effects

The function directly:

- Initializes two local 32-bit temporaries.
- Replaces the second temporary with a record word on the active path.
- Reads the index-like field two or three times on the active path.
- Reads one indexed record word on the active path.
- Reads the guard’s low bit on the inactive path.
- Reads shared values used as dependency arguments and as the inactive-path result.
- Invokes the dependencies in the conditional order described above.

No direct write to the record base, index field, shared result, descriptor, auxiliary values, or guard is visible in this function body.

### Dependency-mediated effects and ownership

Dependencies may mutate data passed through addresses or affect shared state. Their bodies and contracts are not supplied, so this specification does not assert that they preserve globals, records, temporaries, allocation state, or synchronization state.

No allocation, release, ownership transfer, or reference-count adjustment is directly visible. The opaque results and addresses must not be classified as owned, borrowed, retained, or released solely from this caller.

The caller performs no visible cleanup for either temporary or for any value returned by a dependency. Whether cleanup is unnecessary, handled elsewhere, or part of a dependency’s behavior remains unknown.

Addresses of local temporaries are passed to dependencies. The supplied evidence does not establish whether those addresses are retained. Safe use beyond the temporaries’ lifetime would require an external lifetime contract.

## Dependencies

| Opaque ID | Role established by this caller |
|---|---|
| F-8094c3d72052 | Returns the zero/nonzero mode decision. |
| F-81ba24aad07e | Receives writable address-like temporary storage, an indexed record address, and the second input; its result is ignored. |
| F-eac694802fc1 | Receives writable addresses of both temporaries; its zero/nonzero result controls whether the next check runs. |
| F-9316cc774c14 | Receives the current address-like temporary plus `0x14`; zero permits indexed selection, nonzero selects fallback. |
| F-b4a9589b2ea9 | Receives the selected record and second input; supplies the active-path return value. |
| F-b255b26284e6 | Receives the guard reference when its low bit is clear; nonzero permits the guarded sequence. |
| F-9b9c1c7483f4 | Receives the shared result value, descriptor address, and second temporary’s address; supplies the first argument of the next call. |
| F-804adc29bbfb | Receives the preceding result and two shared auxiliary values. |
| F-d08d3ca17b81 | Receives the guard reference after the preceding two guarded calls return. |

The roles “check,” “guard acquisition,” and “guarded sequence” describe call-site structure. They do not establish the dependencies’ internal algorithms or failure semantics.

## Acceptance examples

These are conditional contract examples, not executed tests. All address calculations are assumed valid and representable.

1. **Stable index, both checks zero.**  
   If the mode result is nonzero, the index remains 3, and both checks return zero, the first preparation call receives base + `0x54`. The second temporary is loaded from base + `0x64`. The final processing call receives base + `0x54` and the unchanged second input. Its result becomes the function result.

2. **First check rejects indexed selection.**  
   If F-eac694802fc1 returns any nonzero value, F-9316cc774c14 is not called. F-b4a9589b2ea9 receives base + `0x38`, regardless of the current index.

3. **Second check rejects indexed selection.**  
   If the first check returns zero and leaves the address-like temporary equal to address \(P\), F-9316cc774c14 receives \(P + 0x14\). A nonzero result selects base + `0x38`.

4. **Index changes across calls.**  
   Suppose the initial index is 1, F-81ba24aad07e changes it to 4, and the checks both return zero while changing it to 5. The preparation address is base + `0x1c`; the second temporary is loaded from base + `0x80`; the final selected address is base + `0x8c`. A single cached index would not satisfy this behavior.

5. **Guard already marked.**  
   If the mode result is zero and the guard’s low bit is set, none of the four inactive-path dependencies is called. The final shared result value is returned.

6. **Guard acquisition declines.**  
   If the low bit is clear but F-b255b26284e6 returns zero, the three-call guarded sequence is skipped. The function still returns the shared result, including any change made by the acquisition dependency.

7. **Guarded calls change shared state.**  
   If acquisition returns nonzero, F-9b9c1c7483f4 receives the shared result existing at its call. If later dependencies change that shared result, the final return uses the changed value rather than a saved earlier value.

## Uncertainties

The following are **documented_only**:

- Public ABI details beyond the visible four 32-bit inputs and 32-bit result.
- Record count, index validity rules, alignment requirements, and the meaning of individual record fields.
- Exact machine behavior for overflowing address arithmetic or invalid accesses.
- The guard object’s width, synchronization guarantees, and concurrency protocol.
- Whether shared storage is volatile, atomic, thread-local, or externally synchronized.
- Dependency ownership, allocation, release, retention, and global-preservation contracts.
- Meanings of nonzero check results beyond their visible branch effects.
- Exceptional exits, nonlocal control transfer, or recovery if a dependency fails to return normally.

Inferred intent: the inactive path resembles guarded one-time initialization, and the active path resembles preferred-record selection with a fallback. Neither interpretation adds guarantees about successful initialization, thread safety, or semantic validity of the selected record.

## Evidence

The established behavior derives solely from the supplied static function representation: explicit accesses, argument construction, conditional calls, repeated index loads, and return expressions.

Dependency implementations, external layouts, runtime traces, and calling conventions were not available. No runtime tests have been performed. The acceptance examples describe consequences of the visible control flow under stated conditions; they are not empirical verification.
