<!-- Trusted provenance: F-d87441c1fafd; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-d87441c1fafd

## Purpose

F-d87441c1fafd conditionally stores a supplied unsigned value in a field of a caller-designated object and returns whether its observed acceptance condition succeeded.

The acceptance condition uses a second object reached through the caller-designated object. The second object supplies a nonzero lower bound. The candidate must be at least that bound and strictly below the result of adding `0x80000` to it using unsigned arithmetic.

Two dependency calls surround the function’s own field accesses. The first precedes all object reads; the second follows the conditional store and precedes the return. Their purposes and effects are unknown. In particular, the returned Boolean does not establish that the stored value remains present after the second dependency runs.

This is a source-facing contract based on static evidence only. No runtime tests have been performed.

## Interface

### Inputs

The function receives two arguments:

| Argument | Interpretation |
|---|---|
| Object address | An integer-represented address identifying the primary object. |
| Candidate | An unsigned value tested against the observed bounds and, on acceptance, stored in the primary object. |

The recovered representation uses a signed integer for the address-bearing argument and unsigned integers for the candidate and bound. This contract describes the apparent 32-bit target model: four-byte integer fields, 32-bit unsigned comparison, and unsigned arithmetic modulo `2^32`. The supplied material does not independently establish the platform ABI or compiler type definitions. That qualification is material to arithmetic near the maximum unsigned value.

The object address is used as an address, not compared numerically as a signed quantity. A negative signed representation alone is not an explicit rejection condition.

### Relevant layout

Only these fields are established:

| Object | Byte offset | Access | Meaning within this function |
|---|---:|---|---|
| Primary object | `+4` | Four-byte read | Address-like link to a secondary object; zero means no secondary object is used. |
| Primary object | `+8` | Conditional four-byte write | Destination for an accepted candidate. |
| Secondary object | `+4` | Conditional four-byte read | Unsigned lower bound. |

The primary object’s offset-zero field, if any, is not accessed directly. No complete object size, field naming scheme, allocation origin, or semantic object type is established.

### Output

The function returns a Boolean:

- **True:** the link and bound were nonzero, and the candidate satisfied both unsigned comparisons. The function issued its store of the candidate before calling the final dependency.
- **False:** at least one of those conditions failed. The function did not issue that conditional store.

These meanings apply when execution reaches a normal return. Neither dependency is established to return normally in every circumstance.

The physical representation and calling convention of the Boolean return are not established.

### Preconditions

For execution without invalid memory access:

- After the initial dependency returns, the primary object’s four-byte field at offset `+4` must be readable.
- If the observed link is nonzero, the secondary object’s four-byte field at offset `+4` must be readable.
- If acceptance succeeds, the primary object’s four-byte destination at offset `+8` must be writable.
- The address calculations and accesses must be valid under the actual target’s alignment and memory-access rules.
- The final dependency’s own requirements must be satisfied for the supplied primary-object argument.

There is no explicit null check for the primary object. A zero secondary-object link is handled; a zero primary-object address is not handled by a dedicated rejection branch.

## Behavior

### Observable sequence

On the normal path, the function performs the following ordered actions:

1. Calls F-ccc494c80b38 with no explicit arguments.
2. Reads the primary object’s link field at byte offset `+4`.
3. If that observed link is nonzero, reads the secondary object’s unsigned bound at byte offset `+4`. If the link is zero, it does not perform this secondary-object read.
4. Evaluates acceptance from the observed link, observed bound, and candidate.
5. If acceptance succeeds, writes the candidate to the primary object at byte offset `+8`.
6. Calls F-dd2544d72d08 with the original primary-object argument.
7. Returns the acceptance Boolean.

The initial dependency may change memory before the link is read. Consequently, the acceptance decision uses state observed after that call, not necessarily state present when the function was entered.

The observed link and bound are not reread by the shown function. Changes after those observations do not cause its condition to be reevaluated.

### Acceptance condition

Let the observed unsigned bound be \(B\), the candidate be \(C\), and the unsigned width be 32 bits. Define the upper threshold as:

\[
U = (B + 524288) \bmod 2^{32}.
\]

Acceptance requires all of the following:

- The observed secondary-object link is nonzero.
- \(B \ne 0\).
- \(B \le C\), using unsigned comparison.
- \(C < U\), using unsigned comparison.

The lower comparison is inclusive. The upper comparison is exclusive.

A nonzero link alone is insufficient. A readable secondary object containing a zero bound is explicitly rejected.

### Arithmetic boundaries

For nonzero bounds below `0xFFF80000`, the addition does not wrap. In that case, the accepted candidates form the ordinary half-open interval:

\[
[B,\; B + \texttt{0x80000}).
\]

That interval contains `0x80000` distinct unsigned values.

For bounds from `0xFFF80000` through `0xFFFFFFFF`, the addition wraps. The resulting upper threshold is smaller than the bound. No candidate can then satisfy both the inclusive lower comparison and the exclusive upper comparison. Every candidate is rejected for those bounds.

The behavior is therefore **not** a wraparound range test. It does not accept a candidate merely because its unsigned distance from the bound is less than `0x80000`.

At bound `0xFFF7FFFF`, the upper threshold is `0xFFFFFFFF`. Candidates through `0xFFFFFFFE` can be accepted; `0xFFFFFFFF` is excluded by the strict upper comparison.

### Store and return relationship

On acceptance, the function writes the candidate before invoking F-dd2544d72d08. It does not read the prior destination value or require that the new value differ from it.

The acceptance Boolean is not computed from the destination’s contents after the final dependency. Therefore:

- True does not guarantee that the destination still contains the candidate on return.
- False does not guarantee that the destination is unchanged across the whole call.
- Neither result reports whether either dependency performed other work successfully.

These distinctions follow from the placement of the dependency calls and the absence of any post-call validation.

### Rejection and abnormal execution

Every ordinary rejection path still invokes F-dd2544d72d08. Rejection does not bypass the final dependency.

There is no explicit error code, exception handler, rollback, or recovery path in the supplied body. Invalid object addresses are not converted into a Boolean failure by any shown validation.

If either dependency fails to return normally, or an object access prevents execution from continuing, the supplied body establishes no subsequent normal return behavior. It does not establish whether any earlier dependency effects or writes are reversed.

## Side effects

### Direct memory effects

The function itself has one conditional write: a four-byte store of the candidate at primary-object offset `+8`.

It directly reads:

- The primary-object link at offset `+4`, once.
- The secondary-object bound at offset `+4`, once, only when the link is nonzero.

It does not directly read the existing destination field.

The supplied accesses have no explicit volatile or atomic qualification. The sequence above describes the source-level order of operations; it does not establish hardware transactions, memory barriers, or a concurrency protocol.

### Dependency-mediated effects

Both dependencies may affect state in ways not described by the supplied material. No preservation guarantee is established for globals, object fields, allocation state, locks, or external resources.

The initial call may alter the fields subsequently read. The final call may alter the just-written field or any other accessible state. The function contains no shown restoration of dependency-modified state.

### Ownership and lifetime

The body contains no explicit allocation, deallocation, reference-count update, or ownership-transfer operation.

That absence does not establish the dependencies’ ownership behavior. Callers cannot infer that either dependency retains, releases, or preserves any object.

The primary and secondary storage must remain valid for the accesses actually performed. No lifetime extension is provided by the shown body.

## Dependencies

| Opaque ID | Invocation | Established role |
|---|---|---|
| F-ccc494c80b38 | Once, before the function’s own object reads, with no explicit arguments | Initial dependency whose result is not used by the shown body. |
| F-dd2544d72d08 | Once on every path reaching the final call, after any conditional store, with the original primary-object argument | Final dependency whose result is not used by the shown body. |

Neither dependency’s implementation or contract is available here.

A pairing such as acquire/release, begin/end, or lock/unlock is a possible interpretation, not established behavior. Such an interpretation must not be used to assume mutual exclusion, object validity, memory ordering, or cleanup guarantees.

## Acceptance examples

The following are conditional expectations, not executed tests. They assume the 32-bit model, valid required memory, normal dependency returns, and the stated field values as observed after the initial dependency.

| Observed state and candidate | Expected result | Direct destination write |
|---|---|---|
| Secondary-object link is zero; candidate arbitrary | False | None; no secondary-object bound read |
| Link nonzero, bound `0`, candidate `0` | False | None |
| Bound `0x00100000`, candidate `0x000FFFFF` | False | None |
| Bound `0x00100000`, candidate `0x00100000` | True | Stores `0x00100000` |
| Bound `0x00100000`, candidate `0x0017FFFF` | True | Stores `0x0017FFFF` |
| Bound `0x00100000`, candidate `0x00180000` | False | None |
| Bound `1`, candidate `0` | False | None |
| Bound `1`, candidate `0x00080000` | True | Stores `0x00080000` |
| Bound `1`, candidate `0x00080001` | False | None |
| Bound `0xFFF7FFFF`, candidate `0xFFFFFFFE` | True | Stores `0xFFFFFFFE` |
| Bound `0xFFF7FFFF`, candidate `0xFFFFFFFF` | False | None |
| Bound `0xFFF80000`, candidate arbitrary | False | None; upper threshold wraps to zero |
| Bound `0xFFFFFFFF`, candidate arbitrary | False | None |

Additional sequencing cases:

- **Initial dependency changes the link:** The subsequent read uses the changed link. A pre-call snapshot is not the decision input.
- **Final dependency overwrites an accepted destination:** The function still returns true if execution returns normally; the return records acceptance, not persistence.
- **Final dependency changes the destination after rejection:** The function returns false despite that external change.
- **Destination already equals an accepted candidate:** Acceptance still issues the store. Equality with the old destination is not checked.
- **Nonzero but unreadable secondary-object link:** This is not a defined false result. The body attempts the bound read without a separate validity check.

## Uncertainties

The following are **documented_only**, not filled in by assumption:

- **ABI confirmation:** The apparent 32-bit integer model is not independently verified by target metadata.
- **Address semantics:** Exact pointer representation, valid address ranges, alignment requirements, and behavior of invalid address calculations are unknown.
- **Dependency effects:** Global-state mutation, memory ownership, resource handling, failure behavior, and synchronization are unknown.
- **Object meaning:** The fields could represent addresses, positions, limits, or another application-specific quantity. No semantic label beyond their observed use is established.
- **Aliasing:** The material does not prohibit overlap between the primary and secondary objects. It establishes access order, not disjointness.
- **Concurrency:** There is no established protection against another agent modifying either object during execution.
- **Persistent outcome:** The final destination value and other post-call state cannot be guaranteed without the final dependency’s contract.

The inferred intent is a bounded candidate update bracketed by two support operations. The established behavior is narrower: ordered calls, conditional field accesses, unsigned comparisons, and a conditional store.

## Evidence

Static evidence establishes:

- An initial dependency call before the primary-object read.
- A link read at primary-object offset `+4`.
- A bound read at secondary-object offset `+4`, guarded by a nonzero link.
- Nonzero-link and nonzero-bound requirements.
- An unsigned inclusive lower comparison and strict upper comparison using an increment of `0x80000`.
- A conditional candidate store at primary-object offset `+8`.
- A final dependency call on both acceptance and ordinary rejection paths.
- A Boolean return reflecting the acceptance condition rather than a post-call field check.

The wraparound conclusions follow mathematically under the stated 32-bit unsigned model. No runtime tests, dependency executions, memory-layout probes, or ABI checks have been performed.
