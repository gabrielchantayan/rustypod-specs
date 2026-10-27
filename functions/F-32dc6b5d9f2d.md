<!-- Trusted provenance: F-32dc6b5d9f2d; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-32dc6b5d9f2d

## Purpose

F-32dc6b5d9f2d returns either an existing nonzero value from a caller-supplied structure or the first candidate whose flag byte contains the mask `0x10`.

The existing-value path takes precedence over candidate discovery. It does not validate the existing value against the flag condition. On the discovery path, a zero candidate ends the search and produces zero; a nonzero candidate without the flag causes another candidate to be requested.

**Established static behavior:** the function reads structure fields, calls dependencies, tests candidate flag bytes, and returns a 32-bit value. It contains no explicit memory write.

**Inferred intent:** the structure field at offset `0xF44` may hold a cached or preferred result, and the dependencies may provide an initial candidate followed by subsequent candidates. Those roles are descriptive interpretations, not established ownership or traversal contracts.

## Interface

### Input

The function accepts one value represented as a 32-bit integer in the supplied source. It uses that value as the base address of a structure.

The structure must provide the following readable locations on the paths that access them:

| Offset from input base | Access width | Role |
|---|---:|---|
| `0xF44` | 32 bits | Existing-result field, tested before any dependency call |
| `0xF60` | 32 bits | Argument supplied to F-65fe7bd54a4b when the existing-result field is zero |

The function does not receive an explicit structure length, candidate count, traversal limit, or error-output location.

A nonzero candidate returned by a dependency is used as an address. The function reads one byte at offset `0x1AC` from that candidate and tests mask `0x10`.

### Output

The result is represented as a 32-bit integer:

- On the existing-value path, it is the value obtained by a second read of the field at offset `0xF44`.
- On the discovery path, it is a nonzero candidate whose inspected flag byte has mask `0x10` set.
- On the discovery path, zero means that a dependency returned a zero candidate before a qualifying candidate was selected.

There is no separate error status. Zero does not distinguish an empty candidate sequence from exhaustion after rejected candidates.

The existing-value path has an important qualification: the first read determines whether the branch is taken, but a separate second read supplies the return value. If the field changes between those reads, the returned value can differ from the value that selected the branch, including becoming zero.

### Preconditions and representation limits

Valid readable memory is required wherever an executed path dereferences an address. In particular:

- The existing-result field must be readable to select a path.
- The argument field must be readable only when the existing-result field initially reads as zero.
- Every nonzero candidate that reaches flag inspection must support a readable byte at offset `0x1AC`.

The supplied representation establishes 32-bit field accesses and integer values. It does not independently establish the full calling convention, pointer model, byte order, alignment requirements, or behavior of invalid addresses.

The contract does not promise modular address wraparound or defined signed-overflow behavior. Callers and dependencies must provide values for which the required address calculations and accesses are valid in the actual execution environment.

## Behavior

### Existing-value path

The function first reads the 32-bit field at input-base offset `0xF44`.

If that read is nonzero, the function reads the same field again and returns the second value. On this path:

- The field at offset `0xF60` is not read by this function.
- Neither dependency is called.
- No candidate flag byte is read.
- The returned value need not have mask `0x10` set at any associated address.
- The returned value is not checked for address validity.

For stable memory, the result is the nonzero field value. The two reads must nevertheless remain distinct in the behavioral description; the supplied source does not use the first value directly as the result.

### Discovery initialization

If the initial read of the existing-result field is zero, the function reads the 32-bit field at input-base offset `0xF60`.

It passes that value to F-65fe7bd54a4b in one explicit call. The returned value becomes the first candidate.

The function does not reread the existing-result field during discovery. A dependency’s modification of that field therefore does not, by itself, redirect the remaining selection process.

### Candidate examination

Each candidate is handled according to the following rules:

1. A zero candidate causes an immediate return of zero. No flag-byte read is performed for that candidate.
2. A nonzero candidate causes a one-byte read at candidate offset `0x1AC`.
3. If the byte contains mask `0x10`, that candidate is selected and returned.
4. If the byte does not contain mask `0x10`, the function calls F-986cc6891b07 with no explicit arguments. Its return value becomes the next candidate.

These rules repeat until a qualifying candidate or a zero candidate is obtained.

The test is a mask-membership test, not equality with `0x10`. Other bits may be set without affecting acceptance. For example, flag bytes `0x10`, `0x30`, and `0xFF` all qualify; `0x00` and `0xEF` do not.

A nonzero candidate is not rejected because its 32-bit integer representation is negative. The visible tests distinguish zero from nonzero and inspect the flag byte. Whether a particular nonzero bit pattern is a valid address is an environmental precondition.

### Ordering and termination

The observable sequence on the discovery path is:

- Read the existing-result field.
- Read the initialization-argument field.
- Call F-65fe7bd54a4b.
- For each nonzero candidate, read its flag byte.
- After each rejected candidate, call F-986cc6891b07.
- Stop at the first qualifying candidate or zero candidate.

There is no explicit candidate limit, timeout, cycle detection, or repeated-address detection.

If the dependencies continue to return nonzero candidates whose inspected bytes lack mask `0x10`, the function has no visible termination mechanism. Repeated candidates are inspected again rather than skipped.

Once a candidate qualifies, no additional dependency call is made. A later zero check visible after selection does not introduce a reachable alternate result in the supplied control flow: reaching selection already requires a nonzero candidate, and there is no intervening assignment or call that changes the retained candidate value.

## Side effects

The function itself contains no explicit writes to the input structure, candidate storage, or global storage.

In particular, successful discovery does **not** visibly populate the field at offset `0xF44`. Describing that field as a possible cache does not imply that this function maintains it.

The dependencies may have side effects. Their effects are not established by the supplied material and must not be assumed absent. They could affect global state, traversal state, the input structure, candidate storage, or external resources.

No allocation, deallocation, ownership transfer, retain operation, or release operation is explicit in this function. Returning a candidate does not establish whether the result is borrowed, owned, reference-counted, or valid after subsequent dependency calls.

The function does not visibly release rejected candidates. This observation does not establish a leak: the dependencies’ ownership and lifetime contracts are unknown.

There is no visible recovery for an invalid memory access or a dependency failure outside its ordinary returned value. Exceptions, signals, faults, and other nonlocal effects are undocumented here.

## Dependencies

| Opaque ID | Explicit invocation | Use of result |
|---|---|---|
| F-65fe7bd54a4b | Called once on the discovery path with the 32-bit value read from input-base offset `0xF60` | Supplies the first candidate |
| F-986cc6891b07 | Called with no explicit arguments after each candidate whose flag byte lacks mask `0x10` | Supplies the next candidate |

F-986cc6891b07 is not explicitly passed the previous candidate or the input structure. Any relationship between its result and earlier calls must therefore arise through mechanisms not specified here, such as shared state or an implicit execution context.

The two dependencies are not interchangeable in this contract. The first initializes discovery; the second is used only after a nonzero candidate has been inspected and rejected.

No assumption is made that either dependency preserves globals, preserves candidate contents, owns returned storage, releases previous storage, or provides a finite sequence.

## Acceptance examples

These examples are conditional on valid memory and the stated dependency behavior. They describe expected behavior; no runtime tests have been performed.

### Stable existing result

Given that both reads of the field at offset `0xF44` produce the same nonzero value, the function returns that value without calling either dependency.

This remains true even if the value would fail the candidate flag test. The flag test is not part of this path.

### Existing field changes between reads

Given that the first existing-result read is nonzero and the second read produces a different value, the function returns the second value.

If the second value is zero, zero is returned without entering discovery. Whether such an intervening change is legal or observable in a particular memory model is outside this contract.

### No initial candidate

Given an initially zero existing-result field and F-65fe7bd54a4b returning zero, the function returns zero. It performs no candidate flag-byte read and does not call F-986cc6891b07.

### Initial candidate qualifies

Given an initially zero existing-result field, a nonzero first candidate, and flag byte `0x30` at its offset `0x1AC`, the function returns that candidate. F-986cc6891b07 is not called.

### Rejection followed by selection

Given a first candidate with flag byte `0xEF`, followed by a candidate from F-986cc6891b07 with flag byte `0x10`, the function returns the second candidate. The first byte lacks the required bit despite having other bits set.

### Rejection followed by exhaustion

Given a nonzero first candidate whose flag byte is `0x00`, followed by zero from F-986cc6891b07, the function returns zero. It does not attempt a flag-byte read relative to the zero result.

### Repeated rejection without termination

Given that every continuation result is nonzero and every inspected flag byte lacks mask `0x10`, the function continues requesting and inspecting candidates. No finite return is guaranteed.

## Uncertainties

The following remain **documented_only**:

- The semantic identity of the input structure and its fields.
- Whether the existing-result field is intended as a cache, override, or another kind of stored result.
- The dependencies’ traversal model, state management, error conventions, and side effects.
- The ownership and lifetime of both existing and discovered results.
- Concurrency guarantees, atomicity, synchronization, and whether repeated reads can observe different values.
- Address-space rules, alignment, byte order, and invalid-access behavior.
- Whether dependency results are guaranteed to terminate, remain readable, or avoid cycles.

These unknowns do not alter the visible selection rule, but they limit claims about safety, lifetime, and guaranteed completion.

## Evidence

This contract is based solely on static inspection of the supplied function representation and its supplied dependency-ID mapping.

The static evidence supports the two structure offsets, access widths, candidate flag offset and mask, dependency-call order, separate reads on the existing-value path, zero termination, and absence of explicit writes.

The evidence does not establish dependency implementations, a broader data model, runtime concurrency behavior, or ownership rules. Cache and traversal terminology is inferred intent only.

No runtime execution or tests have been performed.
