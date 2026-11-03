<!-- Trusted provenance: F-c1adec626b62; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-c1adec626b62

## Purpose

F-c1adec626b62 performs an ordered classification of its third input against up to three values obtained through shared state. It uses the opaque dependency F-a1e68ae27737 to determine whether each candidate is selected.

The first dependency result that is nonzero determines the classification:

| Selected candidate | Classification written |
|---|---:|
| First | 2 |
| Second | 3 |
| Third | 4 |

When a candidate is selected, the function obtains a destination pointer from the object designated by its first input and writes the classification into that destination. If all three dependency results are zero, the function returns without performing its own destination-pointer read or classification write.

This is an ordered, first-nonzero selection contract. The supplied evidence does not establish that the dependency performs equality, string matching, validation, or any other particular comparison.

## Interface

The function accepts three inputs and has no specified return value.

| Input | Representation established by the supplied declaration | Role |
|---|---|---|
| First | 32-bit integer used as an address | Designates an object containing a destination-pointer field at byte offset `0x88`. |
| Second | Opaque 32-bit value | Unused by this function. |
| Third | Opaque 32-bit value | Passed as the first argument to every dependency invocation that occurs. |

The relevant memory layout is:

| Memory region | Byte offset | Access width | Meaning within this function |
|---|---:|---:|---|
| Shared candidate storage | `0xD8` / 216 | 32 bits | First candidate supplied to the dependency |
| Shared candidate storage | `0xDC` / 220 | 32 bits | Second candidate supplied to the dependency |
| Shared candidate storage | `0xE0` / 224 | 32 bits | Third candidate supplied to the dependency |
| Object designated by the first input | `0x88` / 136 | 32 bits | Address of the destination object |
| Destination object | `0x6C` / 108 | 32 bits | Classification field written on selection |

A shared base-address slot supplies the address used to locate candidate storage. The function saves a base value before the first dependency call. Later candidate reads use this saved base rather than reloading the shared slot after dependency calls.

The first candidate expression also references the shared base-address slot directly. Thus, the supplied representation establishes a saved base for later candidates and a direct shared-base reference for the first candidate; it does not establish a single atomic snapshot encompassing all candidate accesses.

### Preconditions

For a normally completing invocation:

- Every candidate field actually reached must be readable as a 32-bit value.
- If a dependency result selects a candidate, the first input plus `0x88` must identify a readable 32-bit destination-pointer field.
- The destination address obtained from that field, plus `0x6C`, must identify writable storage for a 32-bit value.
- Every dependency invocation actually reached must satisfy the dependency’s own requirements and return normally.

These conditions are path-dependent. If all dependency results are zero, this function does not dereference the first input. That fact does not establish that arbitrary first-input values are acceptable to the surrounding system.

No null checks, bounds checks, alignment checks, pointer validation, or explicit error-return mechanism are visible.

## Behavior

### Candidate acquisition and selection

The function first captures the shared candidate-storage base for possible use by the second and third candidate reads.

It then reads the 32-bit first candidate at offset `0xD8` through the shared base referenced while preparing the first dependency call. It invokes F-a1e68ae27737 with:

1. The original third input.
2. The first candidate value just read.

A nonzero result selects classification 2. The second and third candidate fields are not read by this function on that path, and no further dependency invocation occurs.

If the first result is zero, the function reads the second candidate at offset `0xDC` from the saved candidate-storage base. It invokes the dependency with the same third input and that newly read candidate.

A nonzero second result selects classification 3. The third candidate field is not read by this function on that path.

If the first two results are zero, the function reads the third candidate at offset `0xE0` from the saved base and invokes the dependency with the same third input and that candidate.

A nonzero third result selects classification 4. A zero third result causes return without the function’s own classification update.

Consequently, assuming normal completion, the dependency is called exactly once, twice, or three times. There is no path with zero dependency calls.

### Result interpretation

The dependency’s result is tested only for zero versus nonzero.

There is no requirement that a successful result equal 1. A negative result, if represented by the dependency’s integer return value, selects the corresponding candidate just as any other nonzero result does.

The function does not distinguish a possible dependency error code from a match result. Whether the dependency has such codes is unknown.

The classifications 2, 3, and 4 are fixed 32-bit values. They are not derived by arithmetic on the candidate data or on the magnitude of the dependency result.

### Destination update

After a candidate is selected, and only after its dependency invocation has completed, the function reads the destination pointer from offset `0x88` of the object designated by the first input.

It then writes the selected classification as one 32-bit value at offset `0x6C` of that destination.

The destination pointer is not cached before the dependency calls. If a dependency changes the pointer field before returning a nonzero result, the function uses the field’s value as read after that call.

The classification field is not read before being overwritten. The function does not preserve its previous value, combine the classification with existing bits, or condition the write on the field’s prior contents.

A successful path performs one explicit classification write, including when the destination already contains the selected value.

### No-selection behavior

When all three results are zero:

- All three candidate fields have been read in order.
- All three dependency calls have completed in order.
- The function does not read the destination-pointer field.
- The function does not perform its classification write.
- It returns without providing a success or failure value.

This is not a guarantee of globally unchanged state. The dependency calls may have side effects, including changes to candidate storage or the destination.

### Width and address arithmetic

Candidate reads, the destination-pointer read, and the classification write are all 32-bit accesses in the supplied representation. The offsets are byte offsets.

The function performs no arithmetic on candidate values. Its relevant address calculations are fixed-offset additions to base addresses.

The evidence does not establish portable behavior for address overflow, invalid mappings, or misaligned accesses. A contract for normal completion therefore requires that every reached address calculation designate suitable storage; it must not treat 32-bit wraparound as a supported addressing feature.

## Side effects

The function’s sole explicit memory write is the 32-bit classification assignment on a selected-candidate path.

Its explicit reads include:

- The shared candidate-storage base references.
- One to three candidate fields, according to earlier results.
- The destination-pointer field, but only after selection.

The function does not explicitly allocate, release, retain, copy, or transfer ownership of any object. Passing an opaque candidate or third-input value to F-a1e68ae27737 does not establish what that dependency does with it. In particular, this function provides no independent assurance that those values remain valid after the dependency returns.

Call boundaries matter:

- Later candidate contents are read after earlier dependency calls, not snapshotted beforehand.
- A change to a later candidate field can therefore affect a later call.
- Changing only the shared base-address slot during a dependency call does not redirect the later reads, which use the saved base.
- Invalidating storage reachable through that saved base may prevent a later read from completing normally.
- A dependency’s change to the destination-pointer field can affect the final write target.

No locking, synchronization, rollback, or recovery mechanism is visible. Concurrent access and reentrant behavior require additional guarantees not established here.

## Dependencies

### F-a1e68ae27737

This is the only explicit function dependency.

| Property | Established behavior |
|---|---|
| Invocation count | One to three, stopping at the first nonzero result |
| First argument | The unchanged 32-bit third input |
| Second argument | A 32-bit candidate read immediately before that invocation |
| Return interpretation | Zero continues the search; nonzero selects the current candidate |
| Side-effect constraints | None established by the supplied evidence |
| Ownership constraints | None established by the supplied evidence |

The function does not establish that this dependency is pure, that it preserves shared globals, that it leaves referenced memory alive, or that it always returns.

If a dependency call does not return normally, the subsequent reads and classification write described for normal completion are not guaranteed to occur.

## Acceptance examples

These examples are conditional expectations derived from static evidence, not runtime test results. They assume valid reached memory accesses and normally returning dependency calls. Unless stated otherwise, the dependency does not introduce additional memory changes.

### First candidate selected

Given that the first dependency call returns any nonzero value:

- Exactly one dependency call occurs.
- Only the first candidate field is read.
- The destination-pointer field is read after that call returns.
- The destination classification becomes the 32-bit value 2.

Results that the second or third call might have produced are irrelevant because those calls do not occur.

### Second candidate selected

Given dependency results of zero followed by nonzero:

- The first and second candidate fields are read in that order.
- Exactly two dependency calls occur.
- The third candidate field is not read.
- The destination classification becomes 3.

### Third candidate selected

Given dependency results of zero, zero, and nonzero:

- All three candidate fields are read in order.
- Exactly three dependency calls occur.
- The destination classification becomes 4.

### No candidate selected

Given three zero results:

- Exactly three dependency calls occur.
- The function performs no destination-pointer read and no classification write.
- Any destination value remains unchanged by this function’s own actions, though dependency side effects remain possible.

### Later candidate changed by an earlier call

Suppose the first dependency call returns zero after changing the second candidate field in storage addressed by the saved base.

The second call receives the changed field value, because that field is read after the first call returns.

### Shared base redirected during a call

Suppose the first dependency call returns zero after changing the shared base-address slot to identify different storage.

The second candidate is still read from the previously saved base. Continued execution requires that storage to remain readable.

### Destination pointer changed before selection completes

Suppose a selecting dependency call changes the destination-pointer field and then returns nonzero.

The final classification write uses the pointer read after that call. It is not bound to the pointer’s value at function entry.

### Existing classification already matches

Suppose the selected classification is 3 and the destination already contains 3.

The function still performs its explicit 32-bit write of 3; it does not first test whether an update is necessary.

## Uncertainties

The following are **documented_only**:

- The application meaning of classifications 2, 3, and 4.
- Whether candidate values and the third input represent pointers, handles, identifiers, or other data.
- The semantic operation performed by F-a1e68ae27737.
- The dependency’s mutation, ownership, lifetime, error, and termination behavior.
- The intended use of the unused second input.
- Whether the shared base can change asynchronously between its initial capture and the first candidate access.
- Platform endianness, alignment rules, fault behavior, and memory-model guarantees.
- The validity of aliased or concurrently modified storage in the surrounding application.

**Inferred intent:** the three candidates appear to represent ordered categories, with the destination field recording the earliest selected category. That interpretation is not required for the established zero/nonzero selection behavior.

## Evidence

The contract is based solely on the supplied static function representation and dependency mapping.

Established evidence includes three ordered dependency-call sites, conditional candidate reads, zero/nonzero branching, a post-selection destination-pointer read, and a 32-bit write of 2, 3, or 4. The second input has no visible use.

No runtime tests have been performed. No dependency implementation, caller contract, complete object layout, or independent platform specification was supplied. Unknown dependency behavior and environmental guarantees remain explicitly unresolved.
