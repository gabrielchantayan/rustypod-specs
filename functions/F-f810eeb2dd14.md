<!-- Trusted provenance: F-f810eeb2dd14; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-f810eeb2dd14

## Purpose

F-f810eeb2dd14 conditionally returns a nonzero candidate obtained from F-abab8af3ce29. Acceptance depends on either a change observed in a caller-supplied integer or a nonzero result from F-578753d1b32d.

If the candidate is nonzero but neither acceptance condition succeeds, the function passes the candidate and a global value to F-ac669dc5909f, then returns zero.

This is an established control-flow contract, not an established resource-management contract. Interpreting the candidate as an allocated object, the integer as a progress counter, or the final dependency as a destructor is inferred intent only. The supplied evidence does not establish those meanings.

## Interface

The function has three supplied arguments and an integer return value:

| Argument | Represented form | Established use |
|---|---|---|
| First argument | Four-byte value, with no established signedness or semantic type | Not read or forwarded by the visible body. |
| Integer-reference argument | Pointer to an integer represented as a signed 32-bit value | Dereferenced on entry, potentially dereferenced again after F-abab8af3ce29 returns, and potentially passed unchanged to F-578753d1b32d. |
| Comparison argument | Integer represented as a signed 32-bit value | Compared for equality with the difference between the second observed integer value and its entry value. |

The return value is represented as a 32-bit integer:

- Zero denotes either failure to obtain a nonzero candidate or rejection of an obtained candidate.
- A nonzero return is the exact candidate returned by F-abab8af3ce29, not the result of F-578753d1b32d.
- No distinct return value identifies which acceptance condition succeeded.

The integer-reference argument must support a valid integer read at entry. The visible function performs no null, alignment, accessibility, or lifetime check before that read.

If F-abab8af3ce29 returns a nonzero candidate, the integer-reference argument must also support the subsequent read. Initial validity alone does not establish validity after an opaque dependency executes. If the secondary dependency is reached, its requirements for that argument are additional and unknown.

The supplied representation uses 32-bit integer quantities and four-byte storage for the first argument. Exact machine arithmetic, pointer width, alignment requirements, and calling-convention details are not independently established by the fragment. In particular, integer-shaped candidate values must not be treated as proven native pointers solely because an offset is added to them.

No structure layout is established except the use of a numeric displacement of 80 from the candidate when preparing one dependency argument.

## Behavior

The following order applies to normal execution, assuming each reached dependency returns normally and each reached memory access and arithmetic operation is valid.

### Entry observation

The function first reads the integer designated by the integer-reference argument and retains that value as the entry snapshot.

This read occurs before F-abab8af3ce29 is called. It occurs even when that dependency will return zero. Consequently, an invalid integer-reference argument is not made safe by a later zero candidate.

The first argument contributes nothing to the visible decision process.

### Candidate acquisition

The function calls F-abab8af3ce29 with no explicit arguments and retains its integer result as the candidate.

If that result is zero:

- The function returns zero.
- It does not perform the second explicit integer read.
- It does not call F-578753d1b32d.
- It does not call F-ac669dc5909f.
- It does not explicitly read the global value used by the rejection path.

The dependency itself may have side effects. A zero result does not establish that the overall operation left memory or global state unchanged.

### Primary acceptance condition

For a nonzero candidate, the function reads the referenced integer again. It compares the comparison argument with the second observed value minus the entry snapshot.

The direction of subtraction matters: this is the later value minus the earlier value. The comparison is exact equality, not a threshold, range, or ordering test.

When the equality holds, the function returns the candidate immediately. It does not call either remaining dependency and does not explicitly read the rejection-path global.

Consequences include:

- An unchanged referenced integer satisfies a comparison argument of zero.
- A decreased referenced integer can satisfy a negative comparison argument when the difference is representable.
- A positive difference does not succeed merely because it exceeds the comparison argument.
- A negative nonzero candidate is not rejected by sign; the visible candidate check distinguishes only zero from nonzero.

There is no explicit overflow check, widening operation, saturation, or range check around the subtraction. For signed 32-bit values whose mathematical difference is not representable, the fragment alone does not establish whether the underlying execution wraps, traps, or follows some other recovered-machine behavior. Such cases require an external arithmetic model.

### Secondary acceptance condition

If the primary equality does not hold, the function calls F-578753d1b32d with:

1. The candidate plus 80.
2. The original integer-reference argument.

The original pointer is passed, not the address of the retained snapshot and not a copy of the referenced integer.

This dependency is skipped entirely when the primary equality succeeds. Its execution is therefore observably conditional.

If F-578753d1b32d returns any nonzero value, the function returns the original candidate. The secondary dependency’s result serves only as a zero/nonzero decision; its magnitude and sign are not propagated.

The visible function does not read the referenced integer again after this call. Any modifications made by the dependency are therefore not subjected to another visible delta comparison.

The addition of 80 has no explicit validity or overflow check. Calling this an access to an embedded member is plausible intent, but no object size, member type, storage extent, or address validity is established.

### Rejection

If the candidate is nonzero, the primary equality fails, and F-578753d1b32d returns zero, the function calls F-ac669dc5909f with:

1. The original candidate.
2. A value read from a global storage location.

That global read belongs to the rejection path and occurs after the secondary dependency returns. The function does not take an entry snapshot of the global. Changes made to it by an earlier dependency can therefore affect the supplied argument.

After F-ac669dc5909f returns normally, the function returns zero. Its return value, if any, is not used by the visible body.

## Side effects

The visible body contains no explicit store through the integer-reference argument and no explicit assignment to global storage. Its direct memory observations are:

- One entry integer read on every execution.
- A second integer read only after a nonzero candidate.
- One global-value read only on the final rejection path.

These reads are separated by dependency calls. The contract does not permit replacing them with an assumption that the referenced integer or global remains unchanged across those calls.

All three dependencies may have additional effects not described by this fragment. Their effects may include changes to caller-visible memory, globals, resource state, or control flow. None can be assumed to preserve globals merely because no explicit arguments name those globals.

Ownership is unresolved:

- The candidate’s origin and resource type are unknown.
- Returning the candidate does not, by itself, establish transfer of ownership.
- Passing the candidate to F-ac669dc5909f does not prove that it is freed, released, or invalidated.
- The function shows no direct allocation or deallocation.
- F-578753d1b32d’s access, retention, or ownership rights over either argument are unknown.

There is no visible error-reporting channel beyond the zero/nonzero return distinction. Zero conflates candidate absence with rejection after secondary evaluation. No rollback is shown for mutations performed before either outcome.

The fragment establishes no locking, atomicity, or synchronization. It does not justify assumptions about concurrent modification of the referenced integer or global.

## Dependencies

| Opaque ID | Invocation condition and arguments | Visible use of result |
|---|---|---|
| F-abab8af3ce29 | Called once after the entry integer read, with no explicit arguments. | Supplies the candidate. Zero ends the function; nonzero enables further evaluation. |
| F-578753d1b32d | Called only for a nonzero candidate whose observed integer delta does not equal the comparison argument. Receives candidate plus 80 and the original integer-reference argument. | Zero selects rejection; any nonzero value selects return of the original candidate. |
| F-ac669dc5909f | Called only after a nonzero candidate, failed primary comparison, and zero secondary result. Receives the original candidate and the rejection-path global value. | No result is used; normal return is followed by a zero return from F-f810eeb2dd14. |

No dependency implementation or independent behavioral contract was supplied. Their names here identify call relationships only.

## Acceptance examples

These are conditional contract examples, not results of runtime tests. They assume valid reached accesses, representable arithmetic, and normal dependency returns.

1. **Zero candidate still requires the entry read.**  
   The referenced integer initially contains 12. F-abab8af3ce29 returns zero. The function returns zero after that call. Neither remaining dependency is called, regardless of the comparison argument.

2. **Exact positive delta bypasses secondary evaluation.**  
   The entry value is 12. F-abab8af3ce29 returns a nonzero candidate and leaves the referenced integer at 17. With comparison argument 5, the function returns the candidate without calling F-578753d1b32d or F-ac669dc5909f.

3. **No change can satisfy primary acceptance.**  
   Both integer observations are 12, the comparison argument is zero, and the candidate is nonzero. The function returns the candidate directly. A positive delta is not required.

4. **A representable negative delta can satisfy primary acceptance.**  
   The entry value is 12 and the later value is 9. With comparison argument −3 and a nonzero candidate, primary acceptance succeeds.

5. **Secondary success returns the candidate, not the secondary result.**  
   Both integer observations are 12 and the comparison argument is 1. The candidate is 4096, so the secondary dependency receives 4176 and the original pointer. If that dependency returns −7, the function returns 4096. The final dependency is not called.

6. **Secondary rejection reaches the final dependency.**  
   Under the same failed-primary conditions, suppose the secondary dependency returns zero. The function reads the rejection-path global, passes its value with candidate 4096 to F-ac669dc5909f, and returns zero after that call returns.

7. **Secondary mutations do not trigger a fresh comparison.**  
   Suppose the primary comparison has already failed. The secondary dependency changes the referenced integer to a value that would have matched the requested delta, but returns zero. The function still follows the rejection path; it performs no visible recheck.

8. **Overflow-boundary subtraction remains undocumented behavior.**  
   If the entry value is the minimum signed 32-bit integer and the later value is the maximum, their mathematical difference exceeds the signed 32-bit range. This contract does not select an acceptance outcome without additional machine-level evidence.

## Uncertainties

The following remain **documented_only**:

- The candidate’s semantic type, validity rules, and ownership.
- Whether candidate plus 80 identifies a field, subobject, separate address, or another encoded value.
- The global value’s type, width, meaning, and synchronization requirements.
- Dependency effects on memory, globals, resource ownership, and argument lifetime.
- Integer overflow semantics and candidate-offset arithmetic at representation boundaries.
- Behavior after invalid memory access or dependency failure to return normally.
- Any higher-level distinction between “no candidate” and “candidate rejected.”

“Inferred intent” is limited to the apparent obtain–evaluate–accept-or-dispose shape. That interpretation must not strengthen the established ownership or cleanup guarantees.

## Evidence

The supplied static fragment establishes the entry snapshot, conditional second read, exact delta comparison, short-circuit dependency ordering, candidate-plus-80 argument, rejection-path global read, and zero-versus-candidate return behavior.

The supplied dependency mapping establishes the opaque IDs used above. No dependency bodies, global declaration, object layout, independent machine specification, or runtime observations were provided.

No runtime tests have been performed. The acceptance examples state consequences of the visible control flow under explicit assumptions; they are not empirical verification.
