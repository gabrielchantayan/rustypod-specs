<!-- Trusted provenance: F-8ec2dc3b7b18; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-8ec2dc3b7b18

## Purpose

F-8ec2dc3b7b18 conditionally submits an operation involving a subject, a selector, and an associated object. It checks an external prerequisite, initializes a temporary context, and applies a threshold. Admission leads to submission, a conditional service notification, an explicit state-byte write, and context finalization.

The two explicit normal-return values are:

- **0** after admission and completion of the subsequent calls.
- **0xFFFFFFD7**, equivalent to **−41** as a signed 32-bit integer, after prerequisite failure or threshold rejection.

“Submission,” “notification,” “context,” and “threshold” identify observable roles. Their application-level meanings, including whether the threshold represents capacity, are inferred rather than established.

## Interface

### Inputs and result

The function accepts three 32-bit arguments:

| Input | Established use |
|---|---|
| **Subject** | An address-like value passed to dependencies and used as a base for field reads. |
| **Selector** | A 32-bit value. Exactly 0xFFFFFFFF, interpreted as signed −1, selects special handling; every other bit pattern selects ordinary handling. |
| **Associated object** | An address-like value passed to dependencies and, on some paths, used for field reads. |

Addresses in the supplied representation are 32-bit. This contract does not establish behavior under a different pointer width.

The return is a 32-bit value. No dependency-derived error is explicitly propagated. In particular, zero does not independently establish application-level success of submission or notification: their results are not checked.

### Observed layouts

All offsets are byte offsets. These are partial access layouts, not complete object definitions.

| Base | Offset | Access |
|---|---:|---|
| Owner returned by F-8ff48652068d | 0xF40 | Read a 32-bit value for the prerequisite; reread it later for context initialization. |
| Owner | 0xF00 | Read a 32-bit address for the completion-state store. |
| Subject | 0xE3C | Read one byte and normalize it to zero or one according to whether it is zero or nonzero. |
| Subject | 0xEF4 | Read an unsigned 32-bit contribution on the matching special-selector path. |
| Associated object | 0x40 | Read a 32-bit address on the nonmatching, nonnull special-selector path. |
| Object reached through associated object + 0x40 | 0x2E | Read an unsigned 16-bit contribution. |
| Associated object | 0x18C | Read one byte; bit 0 controls ordinary-selector mode selection. |
| Object reached through owner + 0xF00 | 0xB92 | Write the single byte value 1 after submission and any notification. |
| Service object | 0 | Read its virtual-table address for dispatch. |
| Service virtual table | 0x1B8 | Read the predicate-method address. |
| Service virtual table | 0x128 | Read the notification-method address when notification is requested. |

The function also supplies a **20-byte temporary storage region** to context-related dependencies. Its internal layout and resource contents are unknown.

These object descriptions do not establish distinct identities or disjoint storage. In particular, the owner-derived completion-state target may overlap storage referenced by either input object.

## Behavior

### Prerequisite and context setup

The function first calls F-8ff48652068d to obtain the owner address. It reads the 32-bit value at owner offset 0xF40 and passes it to F-3b037d560480.

A zero prerequisite result causes immediate return of 0xFFFFFFD7. On this path, the function does not explicitly obtain the service, initialize or finalize the context, read subject or associated-object fields, submit, dispatch a service method, or perform the completion-state store. The two dependencies already called may have their own effects.

For a nonzero prerequisite result, the function:

1. Calls F-87be4aa684d1 and retains the service-object address.
2. Rereads the 32-bit owner field at 0xF40.
3. Calls F-1c06d0a5910b with the temporary storage address, that fresh field value, and zero.
4. Calls F-d5511c118375 with the temporary context to obtain the 32-bit threshold base.

The two owner-field reads are distinct. No preservation of that field across prerequisite checking or service acquisition is established.

### Contribution selection

For the special selector, F-267f68fe54d6 receives the subject. Its 32-bit result is compared for equality with the associated-object argument.

If they match, F-30529d2a9890 receives the subject. Only after that call returns does the function read the unsigned 32-bit contribution at subject offset 0xEF4.

If they do not match:

- A zero associated-object argument gives contribution zero without either fallback dereference.
- A nonzero associated-object argument causes a 32-bit address read at offset 0x40, followed by an unsigned 16-bit read at offset 0x2E of the referenced object. The 16-bit value is zero-extended to 32 bits.

The zero-associated-object exception applies only to the nonmatching branch. If both the dependency result and associated-object argument are zero, equality selects the subject-based branch, including F-30529d2a9890 and the subject contribution read.

For every ordinary selector, the contribution is **1**. No contribution field in the subject or associated object is read for this calculation.

### Threshold and rejection

The threshold calculation adds the base and contribution using **32-bit unsigned arithmetic**, reducing the result modulo 2³².

- A result **greater than 1000** rejects the operation.
- A result **at most 1000** admits it.

The base is represented by a signed 32-bit intermediate, but its use with an unsigned contribution makes this an unsigned addition and comparison. A negative signed interpretation does not independently cause rejection. Overflow is not checked and may produce an admitted result.

On threshold rejection, the function calls F-c1466ea05103 with the temporary context and returns 0xFFFFFFD7. It does not explicitly submit, invoke either service method, or perform the completion-state store.

Special-selector preparation precedes this rejection decision. Consequently, F-267f68fe54d6, and sometimes F-30529d2a9890, may already have produced effects before rejection.

### Submission

On an admitted special-selector path, the function:

1. Reads the byte at subject offset 0xE3C and captures its zero/nonzero Boolean value.
2. Calls F-be0967308708 with the subject to obtain a 32-bit mode.
3. Calls F-d3be6a307196 with the temporary context, subject, selector 0xFFFFFFFF, associated object, mode, and captured Boolean.

The flag is therefore captured before mode selection. Changes to the flag byte during F-be0967308708 do not change this submission’s captured flag.

On an admitted ordinary-selector path, the function first reads the byte at associated-object offset 0x18C:

- If bit 0 is clear, F-be0967308708 receives the subject and supplies the mode.
- If bit 0 is set, the mode is 2, without that mode-selection call.

The function then reads the subject byte at 0xE3C and normalizes it. F-d3be6a307196 receives the temporary context, subject, original selector, associated object, selected mode, and normalized flag.

Thus, on the ordinary dependency-derived-mode branch, the mode call precedes flag capture—the reverse of the special-selector ordering.

No submission result is tested.

### Conditional service notification

After submission, the function reads the service’s virtual-table address, resolves the method at offset 0x1B8, and calls it with the service object.

A zero result skips notification. A nonzero result causes notification preparation.

For the special selector:

1. The subject flag byte at 0xE3C is read again and normalized.
2. F-be0967308708 is called again with the subject to obtain a new mode.
3. The service virtual table is read again, and the method at offset 0x128 is resolved.

For an ordinary selector:

1. The subject flag byte is read again and normalized.
2. The mode previously selected for submission is retained.
3. The service virtual table is read again, and the method at offset 0x128 is resolved.

The notification receives, in order:

1. Service object.
2. Selector, including 0xFFFFFFFF for special handling.
3. Associated-object argument.
4. Selected notification mode.
5. Newly captured Boolean flag.
6. Zero.

Notification flags can differ from submission flags. Special-selector notification modes can also differ because the mode dependency runs twice. Ordinary-selector notification uses the saved mode even if intervening calls alter mode-related object fields.

Notification dispatch uses a fresh virtual-table read. The function does not assume that submission, predicate evaluation, or special-selector mode recomputation leaves the service’s table unchanged. The notification return value is not used.

### Completion-state store and finalization

After submission and any optional notification, the function reads the 32-bit address at owner offset 0xF00. It writes byte value **1** at offset 0xB92 of the referenced storage.

It then calls F-c1466ea05103 with the temporary context and returns zero.

This store occurs even when the service predicate returns zero. Neither explicit rejection path performs it.

The store’s target is owner-derived, but it is not established to be separate from subject or associated-object storage. If the target aliases such storage, this explicit write modifies that input-referenced memory. Object identity and overlap are unknown.

The store precedes finalization. The function therefore establishes that the write occurs, not that the target still contains 1 at return: finalization may change it or affect its lifetime.

## Side effects

### Explicit writes and aliasing

The only explicit nonlocal store shown is the one-byte completion-state write.

No store is syntactically addressed through an input argument. This does **not** establish that input-referenced objects remain unmodified: the owner-derived target may alias the subject, associated object, or other referenced storage. No nonaliasing guarantee is supplied.

Dependencies also receive addresses and may mutate reachable storage. No preservation of globals, fields, virtual tables, or object lifetimes is assumed.

### Context lifecycle and ownership

For dependencies that return normally:

| Path | Context-related calls |
|---|---|
| Prerequisite failure | Neither initialization nor finalization. |
| Threshold rejection | One initialization and one finalization, surrounding threshold preparation. |
| Admission | One initialization and one finalization, surrounding submission, optional notification, and the explicit store. |

The pairing suggests context initialization and cleanup, but it does not prove allocation, release, ownership transfer, or reference retention.

No explicit heap allocation or deallocation appears in this function. Dependencies may perform either. The source does not establish whether any dependency retains the temporary-storage address or other passed addresses.

### Preconditions and invalid access

The function performs no explicit pointer validation. Validity requirements follow the reached path:

- Owner storage must support each reached field read.
- Admitted paths require valid service-object and virtual-table accesses.
- Ordinary admission requires the associated-object mode-byte read.
- A nonmatching special-selector path with a nonzero associated object requires both fallback reads.
- Subject storage must support every reached contribution or flag read.
- Admission requires a valid writable completion-state target.

Required storage must remain valid across preceding dependency calls. A null associated object is not universally supported: only the special-selector nonmatching contribution branch explicitly avoids its dereferences, and admitted dependencies still receive that argument.

No recovery or exceptional cleanup is shown for invalid accesses or dependencies that fail to return normally.

## Dependencies

| Opaque ID or dispatch site | Established call-site role |
|---|---|
| F-8ff48652068d | Returns the owner address. |
| F-3b037d560480 | Receives the first owner-derived value; zero rejects immediately. |
| F-87be4aa684d1 | Returns the service-object address. |
| F-1c06d0a5910b | Receives temporary storage, the reread owner-derived value, and zero. |
| F-d5511c118375 | Supplies the 32-bit threshold base. |
| F-267f68fe54d6 | Supplies the value compared with the associated object for special handling. |
| F-30529d2a9890 | Runs before the matching special-selector contribution read. |
| F-be0967308708 | Supplies mode values with path-dependent timing and repetition. |
| F-d3be6a307196 | Receives the admitted submission and temporary context. |
| F-c1466ea05103 | Receives the context on initialized normal-exit paths. |
| Service dispatch at table offset 0x1B8 | Returns the zero/nonzero notification decision. |
| Service dispatch at table offset 0x128 | Receives the optional notification. |

No opaque function IDs were supplied for the two dynamically resolved methods; their dispatch sites identify them here. Their implementations and ownership behavior are unknown.

## Acceptance examples

These are conditional static acceptance cases, **not runtime test results**. Unless stated otherwise, all reached accesses are valid and all reached dependencies return normally.

1. **Prerequisite failure.** If F-3b037d560480 returns zero, the result is 0xFFFFFFD7. Service acquisition, context initialization, submission, and the explicit completion-state store do not follow.

2. **Ordinary boundary.** A base of 999 produces 1000 after adding 1 and admits submission. A base of 1000 produces 1001, calls finalization, and returns 0xFFFFFFD7 without reading the associated-object mode byte.

3. **Unsigned wraparound.** An ordinary-selector base with bit pattern 0xFFFFFFFF wraps to zero after adding 1 and passes the threshold.

4. **Null special-selector mismatch.** If the associated object is zero and F-267f68fe54d6 returns nonzero, the contribution is zero without fallback dereferences. Admission still depends on the threshold and on valid subsequent calls and accesses.

5. **Null special-selector match.** If the associated object and F-267f68fe54d6 result are both zero, the function calls F-30529d2a9890 and reads the subject contribution. Nullness does not select the zero-contribution branch.

6. **Post-call contribution.** On a special-selector match, a base of 600 and a subject contribution read as 400 after F-30529d2a9890 admit the operation; a contribution read as 401 rejects it. The pre-call field value is not used.

7. **Forced ordinary mode.** If ordinary admission occurs and bit 0 of the mode-selection byte is set, submission uses mode 2. Any notification also uses mode 2, without calling F-be0967308708.

8. **Predicate false.** An admitted submission followed by a zero predicate result skips notification but still performs the completion-state store, finalization, and return of zero.

9. **Special-selector flag timing.** If mode selection changes a previously zero subject flag to nonzero, submission still receives zero. A requested notification captures the flag again before its own mode call.

10. **Mode retention versus recomputation.** On special handling, two different mode results are passed respectively to submission and notification. On ordinary handling, notification retains the submission mode despite intervening object mutations.

11. **Aliasing completion target.** If the owner-derived completion-state byte is the same storage byte as the subject flag, admission writes 1 into that flag after submission and optional notification. This changes input-referenced memory through an owner-derived address. Its value at return remains dependent on finalization.

12. **Fresh completion address.** If an earlier dependency changes owner offset 0xF00, the completion store uses the address read after submission and notification, not a value captured during setup.

## Uncertainties

The following remain **documented_only**:

- Domain meanings of the objects, threshold base, contributions, modes, and completion byte.
- The symbolic meaning of 0xFFFFFFD7.
- Complete object layouts, identities, overlap, lifetimes, and ownership rules.
- Whether dependencies preserve globals, fields, virtual tables, or memory validity.
- Allocation, release, retention, or ownership transfer performed by context-related or other dependencies.
- Internal submission or notification failures and their effects.
- Behavior after invalid memory access or abnormal dependency exit.
- Concurrency, reentrancy, synchronization, and atomicity guarantees.
- Whether unsigned wraparound is intentional and why the threshold is 1000.

Context lifecycle terminology and the capacity-oriented interpretation are inferred intent. Neither implies stronger resource or success guarantees.

## Evidence

The supplied function representation establishes the branches, accesses, widths, argument order, arithmetic, and explicit return values described above. The opaque dependency mapping identifies direct callees; their internal behavior is not available.

In particular, static evidence establishes:

- Two rejection paths with different context lifecycles.
- A 20-byte temporary region.
- Special treatment of selector 0xFFFFFFFF.
- Unsigned 32-bit threshold arithmetic and a strict greater-than-1000 rejection.
- Path-dependent unsigned 32-bit and unsigned 16-bit contributions.
- Different flag-capture and mode-selection ordering by selector class.
- Predicate and notification virtual dispatches with separate table reads.
- An owner-derived one-byte store before finalization.
- No evidence that the store target is disjoint from input-referenced storage.

No runtime tests have been performed.
