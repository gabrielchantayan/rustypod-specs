<!-- Trusted provenance: F-b1e70b9acdbf; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-b1e70b9acdbf

## Purpose

F-b1e70b9acdbf conditionally stores a requested state into a caller-supplied memory region. The operation accepts only the signed 32-bit values 1 and 2, and only when the first 32-bit word of that region equals a separately stored reference word.

The established result is one of two 32-bit status values:

- **0:** the requested state was stored.
- **26**, hexadecimal **0x1A:** an explicit acceptance condition was not satisfied.

The first-word comparison appears intended to recognize an eligible object or record. That interpretation is inferred, not established: the available evidence does not identify the reference value, a record type, or the meanings of states 1 and 2.

## Interface

### Inputs

**Record pointer**

A pointer used to access a region containing 32-bit integer words. Null is explicitly handled and produces status 26.

For a successful operation, the region must support:

| Byte offset from the supplied pointer | Width | Role |
|---|---:|---|
| 0 | 32 bits | Word compared with the reference word |
| 4 | 32 bits | Not accessed by this function |
| 8 | 32 bits | Destination for the requested state |

These offsets describe the accessed layout, not a complete record definition. No evidence establishes the meaning of the intervening word or any fields beyond the destination.

**Requested state**

A signed 32-bit integer. Exactly 1 and 2 are accepted. Every other representable value is rejected.

The accepted value is stored directly as a 32-bit integer; it is not converted into a different state encoding.

### External input

**Reference word**

A separately stored 32-bit value participates in the first-word comparison. Its numerical value, initialization, mutability, and semantic meaning are not supplied.

The comparison is against the value available to this operation, not against a documented literal constant. A caller cannot determine eligibility from the supplied evidence without also knowing that reference value.

### Output

The function returns a 32-bit status value:

| Status | Meaning established by this function |
|---|---|
| 0 | The destination word was assigned the requested state |
| 26 | The pointer was null, the first word did not match, or the requested state was neither 1 nor 2 |

These rejection cases are not distinguished in the return value.

The evidence does not establish a calling convention, pointer width, byte order, or language-level public declaration. The 32-bit widths above describe the supplied integer operations; they do not imply a 32-bit pointer.

## Behavior

### 1. Null rejection

The record pointer is checked before it is dereferenced.

If it is null:

- Return 26.
- Do not read a record word.
- Do not read the reference word through the guarded comparison.
- Do not perform the destination assignment.

The requested state does not change this outcome. In particular, null with state 1 or 2 remains a rejection.

### 2. Eligibility comparison

For a non-null pointer, compare the first 32-bit word of the supplied region with the reference word.

If the values differ:

- Return 26.
- Do not assign the destination word.
- Do not proceed to state acceptance as an additional success condition.

Both comparison operands must be obtained to perform this comparison. The supplied expression does not establish the relative order of the first-word read and the reference-word read. The contract therefore does not require one to precede the other.

This eligibility comparison precedes the state check in the guarded control flow. Consequently, an unacceptable state does **not** make an otherwise invalid non-null pointer safe: the first-word access occurs before state rejection can determine the result.

### 3. Requested-state acceptance

If the pointer is non-null and the eligibility comparison succeeds, accept the request only if its value is 2 or 1.

For every other value:

- Return 26.
- Do not assign the destination word.

There is no evidence of a range rule, masking, truncation, normalization, or acceptance of additional flag bits. Values such as 0, −1, and 3 are rejected rather than mapped onto an accepted state.

### 4. State assignment and success

If all acceptance conditions hold:

1. Assign the requested state to the 32-bit word at byte offset 8.
2. Return 0.

The function does not inspect the destination’s previous value. It therefore accepts an assignment that leaves the stored value unchanged.

There is no arithmetic on the requested state and no integer-overflow case in the state selection or assignment. The destination is a fixed two-word displacement from the supplied pointer, not an input-derived index.

### Memory-access preconditions

A non-null pointer alone does not establish valid memory.

For defined ordinary-memory behavior:

- The first word must be readable whenever the pointer is non-null.
- The reference word must be readable when the eligibility comparison is reached.
- The destination word must be writable when the eligibility comparison succeeds and the requested state is accepted.
- The pointer, alignment, storage lifetime, and pointer arithmetic must satisfy the execution environment’s requirements.

The function does not validate those properties. An inaccessible first word or destination is not a documented status-26 case. The available evidence does not establish fault recovery or a returned status following an invalid memory access.

## Side effects

The only explicit write in the function is the successful 32-bit assignment at byte offset 8. Rejection paths contain no explicit memory write.

The function does not explicitly:

- Allocate or release memory.
- Transfer ownership of the supplied region.
- Retain the pointer for later use.
- Modify the first or intervening word through a separate assignment.
- Read the old destination value.
- Invoke another function.
- Produce diagnostics or update a separate error indicator.

Ownership is therefore not transferred by any visible operation. The caller remains responsible for supplying storage that is valid for the accesses actually reached.

The write must not be interpreted as preserving all separately named storage under every possible aliasing arrangement. If the destination overlaps the storage holding the reference word, the successful assignment can also change that reference word. The evidence supplies no non-aliasing guarantee.

There is no explicit synchronization, atomic operation, or locking. Thread safety and behavior under concurrent mutation are not established. Likewise, the evidence does not define memory-mapped-device behavior or an exact machine-instruction access sequence.

## Dependencies

No callable dependency is visible, and no dependency function ID is supplied.

The operation depends on two data sources:

- The first word of the caller-supplied region.
- The separately stored reference word.

It also depends on writable destination storage for accepted requests.

No unknown callee is involved, so no callee ownership, memory-release, or global-preservation assumption is required. Nevertheless, the reference word’s origin and lifecycle remain unspecified. Its role in the comparison is established; its broader management is documented_only.

## Acceptance examples

These are conditional contract examples derived from static evidence. They are not reports of executed tests.

### Null pointer

Given a null record pointer and requested state 1:

- Return 26.
- Perform no destination assignment.
- Do not attempt the first-word comparison.

The same outcome applies for state 2 or any rejected state.

### Matching first word and state 1

Given valid readable and writable storage, a first word equal to the reference word, and requested state 1:

- Store 1 at byte offset 8.
- Return 0.

The intervening word at byte offset 4 is not accessed by this function.

### Matching first word and state 2

Under the same memory and equality conditions, with requested state 2:

- Store 2 at byte offset 8.
- Return 0.

No additional transition rule depends on the destination’s old value.

### Already-stored state

Given an eligible region whose destination already contains 2, requesting state 2 still reaches the assignment and returns 0. Equality with the previous destination value does not create a rejection case.

### Mismatching first word

Given readable comparison inputs whose values differ, requesting either accepted state returns 26 without the destination assignment.

Destination writability is not exercised on this path. This does not remove the requirement that the initial comparison accesses be valid.

### Rejected state

Given matching comparison words and requested state 3:

- Return 26.
- Do not assign the destination.

The same result applies to 0, −1, and all other signed 32-bit values except 1 and 2.

### Invalid non-null pointer with rejected state

Given an inaccessible non-null pointer and requested state 3, status 26 is not guaranteed. The function attempts the eligibility comparison before rejecting the state. Any resulting access fault is outside the explicit status handling shown.

### Destination aliases the reference word

If the environment permits the destination and reference word to designate the same storage, and the initial comparison succeeds, an accepted request writes that shared storage and returns 0. The earlier comparison is not repeated after the write.

## Uncertainties

The following remain documented_only:

- The numerical reference value and whether it changes between calls.
- The semantic identity of the supplied region and its fields.
- The meanings of states 1 and 2.
- Whether status 26 has a broader named error meaning.
- Pointer width, calling convention, byte order, and concrete alignment requirements.
- Any external restrictions on aliasing or ownership.
- Behavior under races, invalid storage, hardware faults, or unusual memory mappings.
- Exact generated load/store instructions and the relative ordering of the two comparison reads.

The interpretation of the first word as an object-validity marker and the destination as an object state is inferred intent. Neither interpretation is necessary to apply the established acceptance and write rules.

## Evidence

The supplied static function body establishes:

- A null-pointer guard.
- A first-word comparison against separately stored data.
- Acceptance of exactly the signed integer values 1 and 2.
- A single successful assignment two 32-bit words from the supplied pointer.
- Return of 0 after that assignment and 26 on explicit rejection.
- No visible function calls or additional writes.

No implementation execution, runtime tests, binary inspection, or external documentation was available or performed. The contract distinguishes those static observations from inferred purpose and documented unknowns.
