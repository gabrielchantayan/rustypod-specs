<!-- Trusted provenance: F-dd277af4ee6b; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-dd277af4ee6b

## Purpose

F-dd277af4ee6b coordinates an object-selection or object-replacement operation using a mutable primary handle, a required secondary descriptor, an optional tertiary descriptor, and a fourth 32-bit value.

The supplied static representation establishes a sequence of checks, dependency calls, candidate construction, candidate disposal, and state-update requests. It does not establish the application-level meaning of the objects or descriptors. Terms such as “candidate,” “selection,” and “replacement” below describe their roles in this function, not independently verified domain types.

The function has no explicit return value. Its externally meaningful outcomes are dependency calls, any state changes those dependencies perform, and whether processing stops before later operations.

## Interface

### Inputs

The function accepts four arguments:

1. **Primary handle address.** An address from which a 32-bit value is read immediately on entry. This entry value is retained for later operations. The address itself is also passed to several dependencies, which may modify the stored handle or other state.
2. **Secondary descriptor address.** Either null or an address from which a 32-bit value may be read. A null address or a zero first word selects the clearing path.
3. **Tertiary descriptor address.** Either null or an address from which a 32-bit value may be read. A null address or a zero first word enables an early candidate attempt.
4. **Auxiliary value.** An uninterpreted 32-bit value. A local copy is made before the initial guard call. A later dependency receives the address of that copy and can replace it.

The representation uses 32-bit integer and pointer-bearing fields. The exact machine ABI, pointer representation, byte order, and required alignment are not independently established here. The offsets and widths below are properties of the supplied representation, not a portable structure declaration.

### Output

There is no explicit status, object, or error returned by this function. Early termination is silent at this interface unless an invoked dependency has observable effects.

The primary handle address is passed to dependencies capable of performing updates, but this function contains no direct store through that address.

### Preconditions

The primary handle address must be readable for a 32-bit access even if the initial guard would otherwise stop processing. The first read precedes that guard.

A non-null secondary descriptor must be readable for a 32-bit access if execution reaches its check. A non-null tertiary descriptor must be readable for a 32-bit access only if the secondary descriptor passes its check.

Additional memory must be valid on paths that perform nested reads through the saved entry handle. Those requirements are described under Behavior. No local null, bounds, lifetime, or alignment validation protects those accesses.

Aliasing among arguments is not prohibited by an explicit check. Its consequences depend on read timing and dependency effects. In particular, the entry handle is saved, while one later call receives a newly read primary handle.

## Behavior

### 1. Entry capture and guard

The function first reads the 32-bit value at the primary handle address and saves it as the **entry handle**.

It also establishes:

- A **tertiary marker**, initially equal to the original tertiary descriptor address.
- A local 32-bit **auxiliary slot**, initially equal to the fourth argument.

It then calls **F-10ba02f1ed27** without explicit arguments.

Any nonzero result terminates processing immediately. There are no subsequent descriptor reads or update calls on this path. The prior primary-handle read has already occurred, and the guard’s own effects remain possible.

A zero result allows processing to continue.

### 2. Secondary descriptor check

If the secondary descriptor address is non-null, its first 32-bit word is read.

If the address is null or that word is zero, the function calls **F-4bc020038d2e** with:

- The primary handle address.
- A zero value.

It then returns.

No tertiary descriptor word is read on this path. The function does not itself verify that this dependency clears the primary handle; “clearing path” is inferred intent from the call shape.

### 3. Optional early candidate attempt

If the secondary descriptor is present and its first word is nonzero, the tertiary descriptor is considered.

If its address is non-null, its first 32-bit word is read. A non-null address with a nonzero first word skips the entire early candidate attempt and proceeds directly to the general selection phase.

If the tertiary descriptor is null or its first word is zero, the following sequence occurs:

1. The primary handle is read again through its address.
2. **F-e7c4e7dd076d** is called with that newly read handle and the original secondary descriptor address.
3. A zero result terminates the function immediately. It does not fall through to general selection.
4. For a nonzero result, the tertiary marker is changed to null.
5. A one-byte value is read through the saved entry handle:
   - Read a 32-bit pointer-bearing field at entry-handle byte offset 8.
   - From that referenced region, read another 32-bit pointer-bearing field at byte offset 20.
   - Read one byte at byte offset 89 within the second referenced region.
6. **F-4a95613e1504** is called with the entry handle, the byte value, the candidate, the 32-bit all-ones value, and zero.

The byte is a one-byte input, not a 32-bit field read. Its domain meaning and any interpretation beyond that width are unknown.

If F-4a95613e1504 returns nonzero:

1. **F-4bc020038d2e** is called with the primary handle address and the candidate.
2. **F-94cb53877c15** is called with the candidate.
3. The function returns.

If F-4a95613e1504 returns zero:

1. F-94cb53877c15 is called with the candidate.
2. Execution continues into general selection.

Thus, rejection by this early compatibility-like operation allows another approach, while failure to construct the early candidate stops processing.

The saved entry handle and the newly read primary handle need not be equal. The guard or other dependencies may have changed state between the reads.

### 4. General selection

The function calls **F-9c6e06f5ecb8** with five arguments:

1. The primary handle address.
2. The original secondary descriptor address.
3. The original tertiary descriptor address.
4. The address of the local auxiliary slot.
5. The tertiary marker.

The original tertiary descriptor address is passed unchanged even when the marker has been changed to null. These two arguments must not be treated as interchangeable.

The marker is null after an early candidate was successfully constructed but rejected by F-4a95613e1504. If the early attempt was skipped because the tertiary descriptor’s first word was nonzero, the marker remains the original tertiary descriptor address.

The auxiliary slot begins with the fourth argument’s 32-bit bit pattern. F-9c6e06f5ecb8 may overwrite it. The function uses its post-call contents without checking whether a write occurred.

The dependency’s result is interpreted using a signed comparison:

- A negative result terminates processing.
- Zero or a positive result proceeds as a selection index.

Zero is therefore a valid result on this path, unlike the zero-as-failure results of candidate construction.

### 5. Candidate construction and indexed lookup

The function calls F-e7c4e7dd076d again, this time with:

- The saved entry handle.
- The current contents of the auxiliary slot.

This second argument differs from the secondary descriptor address used by the early attempt. Its interpretation is dependency-defined.

A zero result terminates processing.

For a nonzero candidate, the function obtains a 32-bit lookup value by:

1. Reading a pointer-bearing field at byte offset 8 from the saved entry handle.
2. Advancing from that referenced base by 24 bytes multiplied by the nonnegative selection index.
3. Reading one 32-bit word at the resulting address.

Only the first 32-bit word at the selected 24-byte stride is read here. The remaining bytes of any putative entry are not interpreted by this function.

There is no visible upper-bound check on the index. A nonnegative result from F-9c6e06f5ecb8 is not sufficient by itself to establish safe addressing.

The arithmetic is represented using 32-bit quantities. An independent contract cannot assume unlimited-precision multiplication or a valid address for every nonnegative index. Exact overflow and address-formation behavior remain documented_only; valid execution requires an index whose computed access is valid under the actual target semantics.

### 6. Direct resolution or alternate identification

The function calls **F-4e77314e8c36** with the saved entry handle, the candidate, and the lookup value.

If its result is nonzero:

1. **F-85256b82f43d** is called with the primary handle address, that result, and zero.
2. F-94cb53877c15 is called with the candidate.
3. The function returns.

If its result is zero:

1. **F-cf5dd1ff88c3** is called with the saved entry handle, the candidate, and the same lookup value.
2. F-94cb53877c15 is called with the candidate.
3. The alternate identification result determines the remaining path.

For a nonzero alternate result:

1. **F-274b53956040** is called with the primary handle address, zero, and the selection index.
2. **F-2b0966dfc5bf** is called with the primary handle address, the alternate result, and the 32-bit all-ones value.
3. The function returns.

For a zero alternate result, **F-2022b0f73f3b** is called with the primary handle address and a fixed diagnostic indicating that the target object could not be identified. The function then returns.

The diagnostic’s full text, delivery channel, severity, and whether its dependency returns normally are not established.

## Side effects

### Observable ordering

The following orderings are significant:

- The initial primary-handle read occurs before the guard.
- The secondary descriptor is checked before any tertiary descriptor word is read.
- The early candidate is disposed of before general selection continues after rejection.
- The direct-resolution update occurs before candidate disposal.
- Alternate identification occurs before candidate disposal.
- On alternate success, candidate disposal precedes both subsequent update calls.
- The index-based update precedes the update receiving the alternate result.
- On alternate failure, candidate disposal precedes diagnostic reporting.

No guarantee is made about unshown reads or writes inside dependencies.

### State changes

The function directly changes only its local auxiliary slot, marker, saved results, and control state. Externally visible writes are delegated.

The primary handle may change during dependency calls. Most later context-sensitive calls nevertheless continue to use the saved entry handle. The code does not refresh that handle globally after updates.

### Ownership and lifetime

Each nonzero candidate returned by F-e7c4e7dd076d is passed once to F-94cb53877c15 on its visible continuing path.

Calling that dependency is established; actual release, reference decrement, destruction, or ownership transfer is not. In particular:

- Passing a candidate to an update dependency does not establish that the update retains it.
- Candidate disposal after an update does not establish that the update borrowed it.
- The alternate identification result is used after the candidate-disposal call. Its validity must therefore survive that call for the success path to be valid.

The function does not directly dispose of the input descriptors, the auxiliary value, the entry handle, or the nonzero resolution results.

There is no visible exceptional cleanup mechanism. Dependency failure by fault, nonlocal exit, or abnormal termination lies outside the demonstrated normal-return paths.

## Dependencies

All semantics beyond the following call relationships are documented_only:

| Opaque function ID | Role established at this call site |
|---|---|
| F-10ba02f1ed27 | Initial guard; nonzero stops processing. |
| F-4bc020038d2e | Receives the primary handle address and either zero or an early candidate. |
| F-e7c4e7dd076d | Produces a candidate-like value; zero stops the current operation. Called with two distinct second-argument forms. |
| F-4a95613e1504 | Tests the early candidate using the entry handle, one-byte field, all-ones value, and zero. |
| F-94cb53877c15 | Receives each nonzero candidate at the described disposal point. |
| F-9c6e06f5ecb8 | Receives descriptors, an addressable auxiliary slot, and a marker; returns a signed selection result. |
| F-4e77314e8c36 | Attempts direct resolution using an entry handle, candidate, and lookup word. |
| F-85256b82f43d | Receives a nonzero direct-resolution result for an update. |
| F-cf5dd1ff88c3 | Attempts alternate identification after direct resolution returns zero. |
| F-274b53956040 | Receives the selection index on alternate success. |
| F-2b0966dfc5bf | Receives the alternate result and the all-ones value after the index-based update. |
| F-2022b0f73f3b | Receives the fixed diagnostic on alternate identification failure. |

No dependency is assumed to preserve globals, preserve pointed-to state, allocate memory, or release memory.

## Acceptance examples

These are conditional static acceptance cases, not executed tests. They assume valid required memory and dependencies that return normally.

1. **Guard stops execution.** With a nonzero guard result, the primary handle has already been read once. No secondary or tertiary descriptor word is subsequently read, and no later listed dependency is called.

2. **Missing secondary descriptor.** With a zero guard result and a null secondary descriptor, F-4bc020038d2e receives the primary handle address and zero. The tertiary descriptor is not dereferenced.

3. **Empty secondary descriptor.** A non-null secondary descriptor whose first 32-bit word is zero follows the same update-and-return path. No candidate construction occurs.

4. **Early construction fails.** With a usable secondary descriptor and a null or zero-first-word tertiary descriptor, a zero result from the first F-e7c4e7dd076d call ends the operation. General selection is not attempted.

5. **Early candidate accepted.** A nonzero early candidate and nonzero F-4a95613e1504 result cause the candidate update, followed by candidate disposal. General selection is not called.

6. **Early candidate rejected.** A nonzero early candidate and zero F-4a95613e1504 result cause disposal before general selection. That selection receives the original tertiary descriptor argument but a null marker.

7. **Tertiary descriptor bypasses early work.** A nonzero first word at the tertiary descriptor skips early construction and the one-byte nested read. General selection receives the original tertiary descriptor address in both its descriptor and marker positions.

8. **Negative selection result.** A negative result from F-9c6e06f5ecb8 prevents the second candidate construction and indexed lookup.

9. **Index zero.** A zero selection result proceeds. If construction succeeds, the lookup word is read at offset zero from the base obtained through entry-handle offset 8.

10. **Auxiliary replacement.** If general selection replaces the auxiliary slot, the second candidate-construction call receives that replacement, not necessarily the original fourth argument.

11. **Direct resolution succeeds.** A nonzero direct result is passed to F-85256b82f43d with a final zero argument. Candidate disposal follows. Alternate identification is not called.

12. **Alternate resolution succeeds or fails.** After direct resolution returns zero, alternate identification runs before disposal. A nonzero alternate result leads to the index update and then the alternate-result update. A zero alternate result leads instead to diagnostic reporting.

## Uncertainties

- **documented_only — Domain meaning:** The exact meanings of handles, descriptors, lookup entries, and the all-ones argument are unknown.
- **documented_only — Dependency contracts:** Ownership, mutation, error behavior, global-state preservation, and retention of passed addresses are unknown.
- **documented_only — Memory layout:** The representation establishes access widths, offsets, and stride, but not complete structure definitions or bounds.
- **documented_only — Arithmetic:** Exact target overflow, pointer conversion, and address-formation semantics are not independently verified.
- **documented_only — Auxiliary-slot escape:** The address passed to general selection refers to local storage. Whether the dependency retains it is unknown; this function provides no extended lifetime.
- **documented_only — Aliasing and reentrancy:** No explicit prohibition or synchronization is shown. Safety under aliasing, concurrent mutation, or reentrant dependencies is unestablished.
- **Inferred intent:** Candidate “construction,” “disposal,” “resolution,” and state “updates” are role descriptions inferred from call use, not verified implementations.

## Evidence

This contract is based solely on the supplied static representation of F-dd277af4ee6b and the supplied opaque dependency mapping, treated as evidence rather than instructions or trusted semantic documentation.

Established static evidence includes branch conditions, argument ordering, explicit read widths and offsets, the signed negative-result check, the 24-byte indexing stride, call ordering, and normal-return candidate-disposal paths.

No dependency implementations, complete data definitions, runtime traces, or external behavioral documentation were supplied. No runtime tests have been performed. The acceptance examples describe consequences of the visible control flow under their stated conditions; they are not execution results.
