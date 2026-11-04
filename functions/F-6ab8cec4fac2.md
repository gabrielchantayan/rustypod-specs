<!-- Trusted provenance: F-6ab8cec4fac2; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-6ab8cec4fac2

## Purpose

F-6ab8cec4fac2 conditionally updates two 16-bit fields in a referenced state record and, for a restricted set of requested values, invokes a resource-related dependency followed by a configuration-related dependency.

Its established behavior has three main cases:

- A nonzero state flag blocks the update path. The function invokes its finalization dependency and returns the 32-bit value 8.
- With a zero state flag and an accepted requested value, the function writes the requested value, processes and clears a 32-bit resource slot, calls a configuration dependency, updates a second 16-bit field, and returns the configuration dependency’s result.
- With a zero state flag and a rejected requested value, the function skips the configuration sequence but still updates the second 16-bit field, invokes finalization, and returns zero.

Interpreting the two 16-bit fields as a capacity and a remaining-count marker is plausible but not established. This contract describes their byte offsets and arithmetic without requiring that interpretation.

## Interface

### Inputs

The supplied representation exposes three arguments:

| Argument | Representation | Role |
|---|---|---|
| Context address | 32-bit address represented by a signed 32-bit argument | Identifies a context containing a state-record pointer at byte offset `+0x04`. Also passed unchanged to finalization. |
| Requested value | Unsigned 32-bit integer | Tested for eligibility. Its low 16 bits are written to the primary field only when eligible. |
| Adjustment selector | Signed 32-bit integer | A nonnegative value supplies the adjustment directly. Any negative value requests an adjustment derived from the two existing 16-bit fields. |

The signed representation of the context address does not establish that negative address bit patterns are rejected. No address-range validation is visible.

### Return value

The return value is a 32-bit word:

- **8:** the state flag was nonzero when examined.
- **0:** the state flag was zero, but the requested value was rejected.
- **Dependency result:** the state flag was zero and the requested value was accepted; the result comes from F-3452f1df7e41.

These categories need not be numerically distinct. F-3452f1df7e41 may itself return zero or 8.

### Referenced memory layout

Only the following portions of the context and state record are established:

| Object | Byte offset | Access width | Established role |
|---|---:|---:|---|
| Context | `+0x04` | 32 bits | Address of the state record |
| State record | `+0x00` | 32 bits | First argument loaded for F-3452f1df7e41 |
| State record | `+0x15` | 8 bits | Update-blocking flag; zero permits the update path |
| State record | `+0x1C` | 16 bits | Primary field |
| State record | `+0x1E` | 16 bits | Secondary field |
| State record | `+0x60` | 32 bits | Resource-related slot passed to F-94cb53877c15 and then cleared |

Offsets are byte offsets. The supplied representation does not establish byte order, a complete record size, or the meaning of other fields.

### Preconditions

For ordinary completion, the context and referenced state record must support the accesses taken by the selected branch. Dependencies must also return normally.

The initial pointer read occurs before the first dependency call. Consequently, that dependency cannot make an initially invalid context-pointer read safe. It may, however, affect whether the already-captured state address remains usable.

No null checks, allocation-size checks, explicit alignment checks, or synchronization guarantees are established.

## Behavior

### Entry and initial dependency call

The function first reads the state-record address from context offset `+0x04` and retains that address for subsequent direct state accesses.

It then calls F-0b4658e20c99 without explicit arguments. No return value from this dependency is used.

Only after that call returns does the function read the byte at state offset `+0x15`. Therefore, a modification to that byte by F-0b4658e20c99 can affect the selected branch.

The state pointer is not reloaded from the context. If F-0b4658e20c99 changes the pointer stored at context offset `+0x04`, subsequent direct accesses still use the previously captured address.

### Nonzero flag

If the flag byte is nonzero:

1. The function skips adjustment derivation, requested-value processing, resource-slot processing, and both 16-bit field writes.
2. It calls F-96dd78c4ca33 with the original context address.
3. After that dependency returns normally, it returns the 32-bit value 8.

The absence of direct field writes on this branch does not imply that dependencies leave the record unchanged.

### Adjustment derivation

If the flag is zero, the adjustment is determined before any accepted-request write or resource-related call.

For a nonnegative adjustment selector, its signed 32-bit value is used directly.

For any negative adjustment selector, the function reads both fields as unsigned 16-bit values and computes:

**adjustment = primary field − secondary field**

Each field contributes a value from 0 through 65,535. The mathematical difference therefore lies between −65,535 and 65,535 and fits in a signed 32-bit integer.

All negative selector values have the same selection behavior; the original negative magnitude is not used. In particular, the minimum signed 32-bit selector is not negated.

This adjustment is a saved value. Changes made by later dependencies to either field do not cause it to be recomputed.

### Requested-value eligibility

Eligibility is determined from the complete unsigned 32-bit requested value, before narrowing it for storage.

The accepted values are exactly:

- 512
- 1,024
- 2,048
- 4,096
- 8,192
- 16,384
- 32,768

Equivalently, the request must be a power of two in the inclusive range 512 through 32,768.

The supplied arithmetic includes unsigned 32-bit subtraction. Requests below 512 do not become eligible through wraparound. Requests with an accepted low 16-bit pattern but additional high bits are also rejected.

A rejected request does **not** terminate the zero-flag path. It bypasses the following configuration sequence, but the secondary-field update and finalization still occur.

### Accepted-request sequence

For an eligible request, these operations occur in order:

1. The requested value is stored as a 16-bit value at state offset `+0x1C`.
2. The 32-bit value at state offset `+0x60` is read and passed to F-94cb53877c15.
3. After that dependency returns normally, zero is written as a 32-bit value to state offset `+0x60`.
4. The function calls F-3452f1df7e41 with:
   - the 32-bit value loaded from state offset `+0x00`; and
   - the address of the primary field at state offset `+0x1C`.
5. The returned 32-bit word is retained as this function’s result.

There is no visible zero-value guard around the call to F-94cb53877c15. A zero slot value is still passed to that dependency.

The explicit slot clear occurs after F-94cb53877c15 returns. If that dependency writes another value into the slot, the subsequent clear overwrites it. F-3452f1df7e41 executes later and may modify the slot again; a zero slot at function return is therefore not guaranteed.

The primary-field address passed to F-3452f1df7e41 permits that dependency to access the field, but the supplied evidence does not establish its access mode or memory-ownership rules.

### Secondary-field update

On every zero-flag path, after any accepted-request sequence, the function reads the current primary field and writes the secondary field.

The stored 16-bit bit pattern is:

**secondary field = (current primary field − saved adjustment) modulo 65,536**

For a nonnegative selector, only its residue modulo 65,536 affects this final store. There is no clamping or range error.

For a negative selector, the saved adjustment comes from the earlier field values. The final primary-field read happens after the configuration dependency, if invoked. A dependency modification to the primary field therefore participates in the final subtraction.

If the request was rejected, there are no dependency calls between adjustment derivation and this update. Subject to no external interference, a negative selector then causes the secondary field to retain its previous 16-bit value. The write still occurs.

### Finalization and return

After the secondary-field write, the function calls F-96dd78c4ca33 with the original context address.

On normal return from finalization:

- An accepted request returns the saved result from F-3452f1df7e41.
- A rejected request returns zero.

No return value from F-96dd78c4ca33 is used. A nonzero result from F-3452f1df7e41 does not visibly suppress the secondary-field write or finalization.

## Side effects

### Direct writes

The function itself performs only the following established writes to the state record:

| Condition | Direct writes, in order |
|---|---|
| Nonzero flag | None |
| Zero flag, rejected request | Secondary field at `+0x1E`, 16 bits |
| Zero flag, accepted request | Primary field at `+0x1C`, 16 bits; resource slot at `+0x60`, 32 bits, set to zero; secondary field at `+0x1E`, 16 bits |

Dependency side effects are additional and unknown. The table is not a guarantee about the record’s final contents.

### Ownership and lifetime

The function reads and uses a state address supplied through the context. It performs no visible allocation.

Passing the resource-slot value to F-94cb53877c15 and then clearing the slot resembles resource disposal or detachment, but neither release nor ownership transfer is established. In particular, this contract does not assert that the dependency frees memory, closes a handle, or tolerates every possible slot value.

The function does not visibly restore the old primary field or resource slot if F-3452f1df7e41 returns an error-like result.

### Errors and abnormal completion

No local exception handling, rollback, retry, or fault recovery is visible.

The documented ordering describes normally returning calls. If a dependency fails to return, accesses invalid memory, or otherwise terminates execution, later operations are not guaranteed to occur.

The value 8 is established as the blocked-flag return, not as a generally defined error enumeration. Meanings for other return values remain dependency-specific.

## Dependencies

| Opaque ID | Invocation | Established use |
|---|---|---|
| F-0b4658e20c99 | Once after capturing the state pointer, before reading the flag | No explicit arguments; return value unused |
| F-94cb53877c15 | Only for an accepted request on the zero-flag path | Receives the 32-bit resource-slot value; return value unused |
| F-3452f1df7e41 | Only for an accepted request, after the resource-slot clear | Receives the state’s first 32-bit word and the address of its primary field; supplies the eventual result |
| F-96dd78c4ca33 | Once on every normally completing branch | Receives the original context address; return value unused |

These dependencies must not be assumed to preserve globals, preserve referenced memory, or own or release any passed resource. Names such as “finalization” and “configuration” describe their positions in this function, not independently established semantics.

## Acceptance examples

These are conditional, statically derived examples. They are not runtime test results. Unless stated otherwise, dependencies return normally and do not modify the described fields.

### Blocked update

The flag is nonzero when read, regardless of the requested value or adjustment selector.

Expected behavior:

- F-0b4658e20c99 has already run.
- F-94cb53877c15 and F-3452f1df7e41 are not called.
- No direct state-field write occurs.
- F-96dd78c4ca33 is called.
- The return value is 8.

### Accepted request with derived adjustment

Initial primary field: 512. Initial secondary field: 480. Request: 1,024. Selector: any negative value.

The saved adjustment is 32. The primary field becomes 1,024, the resource-slot sequence executes, and F-3452f1df7e41 is called. If that dependency leaves the primary field unchanged, the secondary field becomes 992. Its result is returned after finalization.

### Derived adjustment may be negative

Initial primary field: 512. Initial secondary field: 600. Request: 1,024. Selector: negative.

The saved adjustment is −88, not an unsigned underflow result. If dependencies preserve the primary field, the secondary field becomes 1,112.

### Rejected request still changes the secondary field

Initial primary field: 1,000. Request: 768. Selector: 25.

The request is rejected. The primary field and resource slot are not directly written, and neither resource-related nor configuration-related dependency is called. The secondary field is written with 975. Finalization runs, and the return value is zero.

### Rejected request with derived adjustment

Initial primary field: 1,000. Initial secondary field: 900. Request: 256. Selector: negative.

The saved adjustment is 100. The request is rejected, but the secondary-field write still occurs and stores 900. The return value is zero after finalization.

### Width-sensitive adjustment

Current primary field at the final read: 1,024. Selector: 65,537.

The secondary field becomes 1,023 because the stored result is reduced modulo 65,536. The selector is not rejected for exceeding the field width.

### Dependency modifies the primary field

Initial primary field: 512. Initial secondary field: 480. Request: 1,024. Selector: negative.

The saved adjustment is 32. If F-3452f1df7e41 changes the primary field to 2,048 before returning, the subsequent secondary-field write stores 2,016, not 992.

### Eligibility boundaries

With a zero flag:

- 512 and 32,768 are accepted.
- 0, 256, 768, 65,536, and 4,294,967,295 are rejected.
- 66,048 is rejected even though its low 16 bits equal 512.

Every rejected case still reaches the secondary-field update and finalization under normal execution.

## Uncertainties

The following remain **documented_only**:

- Semantic meanings of the context, flag, two 16-bit fields, first state word, and resource slot.
- Whether the initial and final dependencies implement locking, bookkeeping, or another protocol.
- Dependency return conventions beyond the observed propagation.
- Resource ownership, resource validity requirements, and release behavior.
- Whether F-3452f1df7e41 modifies the primary field or other memory.
- Byte order, complete object layouts, alignment requirements, and address-space restrictions.
- Concurrency guarantees and visibility to other execution agents.
- Behavior after invalid memory access or abnormal dependency completion.

Inferred intent is limited to a guarded reconfiguration operation that preserves a previously derived difference when the selector is negative. That interpretation is not required for conformance to the established reads, writes, calls, and arithmetic.

## Evidence

This contract is derived solely from static inspection of the supplied source representation. That representation is treated as evidence to analyze, not as authoritative commentary or behavioral instructions.

Established evidence includes access widths and offsets, branch conditions, dependency-call placement, direct-write order, unsigned eligibility arithmetic, adjustment derivation, and return selection.

The accepted-value set and modulo-65,536 update rule are consequences of that static arithmetic. Resource disposal, synchronization, and domain-specific field meanings are not established.

No runtime tests have been performed. The acceptance examples specify conditional expected behavior; they do not claim observed execution.
