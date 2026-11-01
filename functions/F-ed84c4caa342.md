<!-- Trusted provenance: F-ed84c4caa342; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-ed84c4caa342

## Purpose

F-ed84c4caa342 searches a sequence of address-like records for a record whose comparison field equals a supplied integer key. It returns the matching record’s address-like value, or zero when the search ends without an accepted match.

The search treats the initial record differently from later records:

- The initial record is compared immediately, without a continuation check.
- Each subsequent record is obtained through a dependency, processed by another dependency, and admitted for comparison only if a continuation equality holds.
- A subsequent record can therefore contain the requested key and still be rejected without its comparison field being read.

This is the established control-flow behavior. Interpreting the sequence as a particular container, or the continuation equality as a particular boundary or classification rule, is not established.

## Interface

### Inputs

The function has four input positions:

| Position | Contract role |
|---|---|
| First | Address-like value identifying a containing object whose fields supply the initial record, traversal context, and continuation value. |
| Second | Integer search key, compared for exact equality with a record field. |
| Third | A four-byte value that is not read or forwarded by the supplied function body. |
| Fourth | Initial value of a local integer state cell that a dependency may subsequently modify. |

The supplied representation uses integer-typed values both for scalar integers and for addresses. Its field accesses and the explicitly four-byte third argument support a recovered 32-bit data model. The original ABI, pointer representation, alignment requirements, and machine-level arithmetic rules have not been independently established.

The behavioral descriptions below use four-byte fields in that recovered model. They do not imply that the containing objects consist only of the fields listed.

### Containing-object layout

Offsets are byte offsets from the first input’s address-like value.

| Offset | Width | Use |
|---|---:|---|
| +4 | 4 bytes | Context argument loaded immediately before each call to F-86f132defe26. |
| +20 | 4 bytes | Initial record value, loaded once on entry. Zero denotes an empty starting point. |
| +24 | 4 bytes | Continuation value, loaded after each returning call to F-3bcbe089f356. |

### Record layout

| Offset | Width | Use |
|---|---:|---|
| +8 | 4 bytes | Integer field compared with the second input. |

No other record fields are directly accessed by this function. Dependencies may access additional storage.

### Output

The return value is either:

- The current record’s address-like value, when its field at offset +8 equals the search key during an admitted comparison.
- Zero, when the initial record is zero, traversal returns zero, or a continuation equality fails.

There is no separate status output or error code. A zero result does not distinguish these termination reasons.

### Preconditions

For defined execution, every directly performed memory access must be valid at the moment it occurs. In particular:

- The initial read at containing-object offset +20 must be valid even if the value stored there is zero.
- Every admitted nonzero record must support the read at offset +8.
- A transition requires a valid read at containing-object offset +4.
- A returning nonzero transition, followed by a returning processing call, requires a valid read at containing-object offset +24.

A zero first input is not explicitly checked. A nonzero address-like value is not proof of valid memory.

Dependency-specific validity, lifetime, and context requirements are documented_only because their implementations are not supplied.

## Behavior

### Entry and initial comparison

The function reads the initial record from containing-object offset +20 and creates a local integer state cell initialized from the fourth input.

If the initial record is zero, the function returns zero. It does not call either dependency, read the continuation field, or inspect a record.

If the initial record is nonzero, the function reads its field at offset +8 and compares it with the second input using integer equality.

A match returns that record immediately. In this case:

- Neither dependency is called.
- The containing-object context field at +4 is not read.
- The continuation field at +24 is not read.
- The fourth input does not restrict acceptance of the initial record.

The initial comparison is therefore unconditional with respect to the continuation rule.

### Transition after a mismatch

After an admitted record fails the key comparison, the function reads the containing-object field at offset +4. It calls F-86f132defe26 with:

1. That freshly loaded context value.
2. The current record value.

The returned value replaces the current record.

If that return value is zero, F-ed84c4caa342 returns zero immediately. It does not call F-3bcbe089f356 and does not evaluate the continuation equality for this transition.

If the returned value is nonzero, the function calls F-3bcbe089f356 with:

1. The newly returned record value.
2. The address of the local integer state cell.

The local state cell is initialized only once. It is not reset from the fourth input before later calls. Changes made through the supplied address can therefore carry across transitions.

The processing dependency’s return value is not consumed. This does not imply that the dependency has no observable effects.

### Admission of a subsequent record

After F-3bcbe089f356 returns, the function compares:

- The current value read from containing-object offset +24.
- The local state value plus one.

If the equality holds, the newly obtained record is admitted to the next key comparison.

If the equality fails, the function returns zero without reading that record’s comparison field.

Consequently, processing a nonzero record through F-3bcbe089f356 does not imply that the record will be searched. Processing precedes admission, and admission precedes the key comparison.

The continuation field is not cached at entry. Its read occurs after the processing call, so changes to that field made by dependencies can affect the decision, provided the memory remains valid.

### Repetition and stopping

For each admitted record:

- An equal comparison field ends the search successfully.
- A different comparison field causes another transition.
- A zero transition result ends the search unsuccessfully.
- A nonzero transition result is processed before its continuation equality is checked.
- A failed equality ends the search unsuccessfully.

There is no explicit iteration limit, cycle detection, or record-uniqueness check. If dependencies repeatedly produce nonzero records, every continuation equality succeeds, and no compared field matches, the function need not terminate.

The initial record field at containing-object offset +20 is not reloaded. Changes to that field after entry do not directly replace the current record.

### Arithmetic and equality

Both record-key matching and continuation checking use equality, not ordering.

The continuation expression adds the integer value one to the current local state. For values where this addition is representable in the recovered signed integer model, the contract is ordinary integer equality.

The supplied representation does not establish a portable result for signed overflow. In particular, this contract does not assume wrapping when the local state is the maximum signed 32-bit integer. Machine-level behavior for that boundary is documented_only pending stronger evidence.

No normalization, masking, or conversion of the search key is visible in the function body.

## Side effects

The function directly performs reads from the containing object and admitted records. It directly initializes a local state cell, but contains no explicit writes to the containing object or records.

Its externally consequential actions include the dependency calls. Those calls must not be assumed to preserve:

- The containing object or its fields.
- Record contents or validity.
- Global state.
- Allocation ownership.
- Other externally visible resources.

In particular, the function reads the context field anew for every transition and reads the continuation field after processing. Substituting entry-time snapshots would not preserve behavior when dependencies change those fields.

The local state cell is shared with F-3bcbe089f356 by address. The storage is local to this invocation and its lifetime ends when the invocation ends. Whether the dependency retains that address is unknown; no retention or post-return validity guarantee is established.

There is no explicit allocation, deallocation, ownership transfer, or resource release in the supplied body. Returning a record does not establish that the caller owns it, that it is newly allocated, or that it remains valid after return. Ownership and lifetime effects inside dependencies remain unknown.

No explicit recovery handles invalid memory, a nonreturning dependency, or a dependency-raised failure. Ordinary execution continues only when each required access and call completes.

## Dependencies

### F-86f132defe26

**Observed invocation:** receives the current containing-object context field and the record whose key comparison just failed.

**Observed result use:** its return becomes the next current record. Zero terminates the search immediately; a nonzero value is passed to F-3bcbe089f356.

**Inferred intent:** advances or otherwise selects another candidate record.

**Unknowns:** selection order, handling of cycles, allocation behavior, mutation, errors, ownership, and preservation of containing-object storage.

### F-3bcbe089f356

**Observed invocation:** receives a nonzero record returned by F-86f132defe26 and the address of the invocation’s local integer state cell.

**Observed result use:** no return value is consumed. After return, the local state participates in the continuation equality.

**Inferred intent:** derives or updates state associated with the newly selected record.

**Unknowns:** whether it writes the state on every call, which values it may write, other memory effects, failure behavior, and address-retention behavior.

Neither dependency’s inferred purpose is required to describe the caller’s established control flow.

## Acceptance examples

These examples are conditional contract cases, not executed tests. Unless specified otherwise, required memory is valid, dependencies return normally, and arithmetic does not overflow.

### Empty initial record

The containing object’s field at +20 is zero.

**Expected:** return zero after reading that field. Neither dependency is called. The context and continuation fields are not read.

### Immediate match despite an incompatible continuation value

The initial record is nonzero and its field at +8 equals the search key. The containing object’s continuation value would not equal the initial local state plus one.

**Expected:** return the initial record. The incompatible continuation value is irrelevant and is not read.

### Transition reaches zero

The initial record does not match. F-86f132defe26 receives the current context and that record, then returns zero.

**Expected:** return zero. F-3bcbe089f356 is not called, and the continuation field is not read for this transition.

### Later record is admitted and matches

The initial record does not match. F-86f132defe26 returns a nonzero second record. F-3bcbe089f356 leaves the local state equal to 6. The subsequent read at containing-object offset +24 yields 7. The second record’s comparison field equals the key when read.

**Expected:** return the second record. Its comparison field is read only after processing and successful continuation checking.

### Later matching record is not admitted

A transition returns a nonzero second record whose comparison field contains the key. Processing leaves the local state equal to 6, but the continuation field reads as 8.

**Expected:** return zero without reading the second record’s comparison field. Its stored key cannot override failed admission.

### State persists across transitions

The first processing call changes the local state to 6, and the continuation field reads as 7. The admitted record does not match. A second transition returns another nonzero record.

**Expected:** the second processing call receives the address of the same local state cell, containing 6 if nothing has subsequently changed it. The fourth input is not reapplied.

### Dependency mutations affect later decisions

A dependency changes the context field before a later transition. A processing call changes the continuation field before returning.

**Expected:** the later transition uses the newly read context, and the continuation decision uses the continuation value read after processing. Neither value is fixed by an entry-time snapshot.

### Third input varies

Two otherwise equivalent executions differ only in the third input.

**Expected:** the supplied body performs no different reads, calls, comparisons, or explicit writes because of that input. This is a body-level observation, not a claim about unspecified external instrumentation.

## Uncertainties

The following are documented_only rather than guessed:

- Original ABI details, including independently confirmed integer and pointer widths, alignment, and address representation.
- Full object layouts and semantic names for the accessed fields.
- The dependencies’ traversal and processing rules.
- Dependency effects on globals, memory validity, ownership, and resource lifetimes.
- Behavior at signed-overflow boundaries in the continuation addition.
- Concurrency, synchronization, and whether any accessed storage is volatile.
- Domain-specific meaning of a zero result.
- Any guarantee of termination or uniqueness of visited records.

The function’s shape suggests a constrained search, but the meaning of its constraint is not established. No runtime tests have been performed.

## Evidence

The supplied static body establishes the field offsets, direct read order, initial-record exception, dependency call order, persistent local state cell, post-processing continuation check, and return branches described above.

The dependency map establishes the opaque dependency identities used here. No dependency implementation was supplied, so their semantics remain limited to observable call arguments and result handling.

The distinction between established behavior and inferred intent is deliberate: candidate selection and state derivation are descriptive interpretations, while the reads, calls, comparisons, and branch outcomes are directly supported by the supplied representation.
