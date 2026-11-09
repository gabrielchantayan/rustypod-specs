<!-- Trusted provenance: F-da3be2b0546d; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-da3be2b0546d

## Purpose

F-da3be2b0546d coordinates a mask-selected operation on an indexed entity within an owner context. For a nonzero accepted mask, it notifies selected recipients, performs conditional and unconditional preparation, submits a fixed-size record, waits for a dependency to return a nonzero result, and updates owner, table, and subsystem state.

The interpretation as entity teardown or deactivation is **inferred intent**, not an established interface guarantee. The visible behavior establishes the calls, arguments, ordering, and direct memory changes described below; it does not establish the meanings of the dependency operations.

A zero mask is a no-op. A mask containing any prohibited bit enters a dependency identified as non-returning.

## Interface

### Inputs

The function accepts three 32-bit arguments:

| Input | Representation | Use |
|---|---|---|
| Owner context | Address carried in a 32-bit argument | Passed to dependencies and used for a direct four-byte field access at byte offset `0x2D0`. |
| Entity identifier | 32-bit integer | Passed to dependencies, placed in the submitted record, compared with the owner field, and used to index a global table. |
| Selection mask | Unsigned 32-bit integer | Validated, used to select recipient notifications, and placed unchanged in the submitted record. |

The source establishes neither an entity-identifier range nor a null-address convention. It contains no local validation of the owner address, entity range, dependency results used as addresses, or table bounds.

### Return and completion

The function has no explicit result value.

Normal completion occurs either immediately for a zero mask or after the full nonzero-mask sequence. Completion is not guaranteed: the invalid-mask dependency is marked non-returning, and the wait loop has no visible retry limit or timeout.

### Mask domain

A mask is accepted exactly when:

**mask AND `0xFFFFE040` equals zero.**

Equivalently, only bits 0–5 and 7–12 may be set; the complete allowed-bit mask is `0x00001FBF`.

Bit 0 is accepted but does not select a recipient notification. It still makes the mask nonzero and therefore enables the complete nonzero-mask sequence. Bit 6 is prohibited. Every bit from 13 through 31 is prohibited.

### Memory preconditions

For a nonzero accepted mask, the environment must provide valid storage and callable dependency targets for every reached access. In particular:

- The owner must support a four-byte access at offset `0x2D0`.
- The context accessor must return values suitable for the subsequent address adjustment and dependency calls.
- Each selected recipient lookup must yield an object whose first word identifies a dispatch table with a callable entry at byte offset `0x0C`.
- The global entity table must support the indexed byte access described below.
- Dependencies receiving addresses of local storage must obey whatever lifetime contract applies to those addresses.

The function does not check these preconditions.

## Behavior

### 1. Validate before all operational work

The mask check precedes recipient lookup, record submission, owner-field access, and table access.

If any prohibited bit is set, F-a817d8d58c10 is called without arguments. The supplied source marks this call as non-returning. No ordinary return or recovery path is established for this case.

If the accepted mask is zero, the function returns immediately. No dependency call or direct external-memory write is visible on this path.

### 2. Notify selected recipients in ascending order

For each position from 1 through 12, in ascending numerical order, the function tests the corresponding mask bit.

For each selected position:

1. It calls an unmapped context accessor.
2. It adds four bytes to the returned context value.
3. It calls F-9266831caef7 with that adjusted context and the selected position.
4. It dispatches through the returned object's dispatch-table entry at byte offset `0x0C`, passing the object, the constant `8`, and the entity identifier.

The recipient-position argument is converted through an eight-bit character representation. Because the positions are only 1–12, this conversion does not change their numerical values regardless of character signedness.

The accessor is called separately for every selected position. A contract must not replace those calls with one cached context lookup: intervening dependencies could change the accessor's result or related global state.

No recipient is dispatched for bit 0. Bit 6 cannot be selected on a normally proceeding path because validation rejects it.

### 3. Perform classification-dependent preparation

After all selected-recipient calls:

1. The context accessor is called again.
2. F-c1e400b68436 receives the adjusted context and entity identifier.
3. Only if its result is exactly decimal `19`, F-a2715f79cdcc is called with the entity identifier.
4. If that result, interpreted as unsigned 32-bit, is less than `2`, F-352a899fb784 is called with the owner and that result.

Thus the conditional call to F-352a899fb784 can receive only `0` or `1` on this path. A result with its high bit set does not qualify merely because it might represent a negative signed integer.

F-88707f7d96dd is then called with the owner and entity identifier, whether or not the classification-dependent call occurred.

### 4. Construct and submit a 20-byte record

The function initializes local storage with this layout:

| Byte offset | Width | Initialized content |
|---|---:|---|
| `0x00` | 1 byte | `7` |
| `0x01`–`0x03` | 3 bytes | Not explicitly initialized in the visible source |
| `0x04` | 4 bytes | Decimal `16` |
| `0x08` | 4 bytes | Entity identifier |
| `0x0C` | 4 bytes | Result of F-1bc5bcdc9cca |
| `0x10` | 4 bytes | Original selection mask |

To obtain the field at offset `0x0C`, the function makes another fresh context-accessor call, then calls F-1bc5bcdc9cca with the adjusted context and entity identifier.

It submits the record to F-c6c589453fa8 with arguments `0`, `0`, the record address, and byte length `20`.

The record is not established to be a portable serialization. In particular, the visible source does not establish byte order or values for offsets `0x01`–`0x03`. The contract must not promise those bytes are zero.

### 5. Wait until a dependency returns nonzero

After record submission returns, the function reads a global four-byte seed once.

For each wait attempt, it initializes two distinct local four-byte cells to that captured seed and calls F-704f99fd5e22 with:

- The owner.
- The entity identifier.
- The address of the first cell.
- The address of the second cell.
- A final argument of zero.

A zero return causes another attempt. Any nonzero return ends the loop, including a value that would be negative under signed interpretation.

Both cells are reset before every attempt. Values written into them by an earlier attempt do not carry into the next attempt. The global seed is not visibly re-read between attempts, and the cells' resulting values are not otherwise used by this function.

There is no visible delay, cancellation check, retry bound, or separate error-result branch.

### 6. Conditionally clear the owner's entity field

After the wait terminates, the function reads the four-byte owner field at offset `0x2D0`.

If it equals the entity identifier:

1. The function writes zero to that field.
2. It examines the record's four-byte field at offset `0x04`.
3. If that field is not decimal `11`, it calls F-fa1e7c5a0d15 with argument `1`.

If the owner field does not match, neither the clear nor this notification occurs.

The record field was initialized to `16`, so absent intervening modification it differs from `11`. Nevertheless, its comparison occurs after external calls, including a call given the record's address. Treating it as permanently constant would require additional dependency guarantees not supplied here.

The owner-field clear precedes F-fa1e7c5a0d15.

### 7. Preserve or clear an indexed status byte

The function obtains a global table base and computes a record location using an entity stride of `0x24` bytes. It examines the byte at offset `4` within that record.

The byte test distinguishes the raw value `0xFE` from every other value:

- If the byte is `0xFE`, the function calls F-ca1c4e2f14bb with the owner, entity identifier, and false, then writes `0xFE` to the previously computed byte address.
- Otherwise, it calls F-ca1c4e2f14bb with the owner, entity identifier, and true, then writes zero to that address.

The decision and destination address are established before F-ca1c4e2f14bb runs. Its changes to the byte or global table-base value are not re-read before the direct write. Consequently, the function overwrites any intervening change to that particular byte with the value selected from the earlier observation.

No bounds or arithmetic-overflow checks are visible for the indexed address calculation. The source uses a 32-bit entity value and arithmetic involving a 36-byte stride; it does not establish a safe entity domain or portable overflow semantics.

### 8. Perform final subsystem operations

The function then makes these calls in order:

1. F-1cd96d01ddae with adjusted context, entity identifier, and `0`.
2. F-9ef7726bf517 with adjusted context, entity identifier, and `0`.
3. F-39405f5827ef with adjusted context and entity identifier.
4. F-4944ce526800 with adjusted context, entity identifier, `0`, and `0`.

Each operation is preceded by a separate context-accessor call. Context identity across the four operations is not guaranteed by this function.

After the fourth operation returns, the function returns.

## Side effects

### Direct external-memory writes

On the nonzero accepted-mask path, direct externally visible writes are limited to:

- A conditional four-byte zero write to owner offset `0x2D0`.
- A one-byte write to the indexed table record at offset `4`, after F-ca1c4e2f14bb returns.

Dependencies may have additional effects. Their names, arguments, or apparent usage do not establish restrictions on global mutation, callbacks, allocation, or resource release.

### Local storage and ownership

The function creates a 20-byte submission record and two four-byte wait cells in local storage. It passes addresses of that storage to dependencies.

No explicit allocation, deallocation, ownership transfer, or reference-count operation is visible in this function. It is unknown whether any dependency allocates or releases resources associated with the owner, entity, recipient object, or other state.

The source alone does not prove that the submission dependency copies the record synchronously or that dependencies refrain from retaining local addresses. Such retention would require an independently established lifetime arrangement; none is supplied here.

### Error handling and partial effects

Except for the mask check and wait-loop result test, the function does not visibly interpret dependency results as errors.

There is no rollback sequence. If a dependency fails to return after prior notifications or state changes, those earlier effects are not locally undone.

## Dependencies

| Opaque ID | Established use |
|---|---|
| F-a817d8d58c10 | Non-returning invalid-mask path. |
| F-9266831caef7 | Selected-recipient lookup using adjusted context and position. |
| F-c1e400b68436 | Produces the classification compared with `19`. |
| F-a2715f79cdcc | Produces the unsigned value tested against `2`. |
| F-352a899fb784 | Conditional owner operation receiving `0` or `1`. |
| F-88707f7d96dd | Unconditional preparation for every accepted nonzero mask. |
| F-1bc5bcdc9cca | Supplies the record's four-byte field at offset `0x0C`. |
| F-c6c589453fa8 | Receives the 20-byte local record. |
| F-704f99fd5e22 | Repeated wait operation; zero repeats, nonzero completes. |
| F-fa1e7c5a0d15 | Conditional notification after clearing the matching owner field. |
| F-ca1c4e2f14bb | Receives the Boolean derived from the table byte before its replacement. |
| F-1cd96d01ddae | First final subsystem operation. |
| F-9ef7726bf517 | Second final subsystem operation. |
| F-39405f5827ef | Third final subsystem operation. |
| F-4944ce526800 | Fourth final subsystem operation. |

Two additional dependency identities are **documented_only**: the repeated context accessor and the indirect recipient-dispatch target. No opaque IDs for them are supplied. Neither is assumed pure or globally state-preserving.

## Acceptance examples

These are conditional behavioral criteria, not executed tests.

1. **Zero mask:** With mask `0`, the function returns without calling dependencies or accessing the owner field or indexed table.

2. **Accepted bit without recipient dispatch:** With mask `0x00000001`, no recipient-dispatch call occurs. All subsequent nonzero-mask stages still occur, subject to dependencies returning.

3. **Ascending recipient order:** With mask `0x00001082`, recipient positions `1`, `7`, and `12` are dispatched in that order. Each receives operation argument `8` and the supplied entity identifier.

4. **Prohibited bit:** With mask `0x00000040`, the invalid-mask dependency is entered before any recipient lookup or ordinary nonzero-mask work. The same requirement applies to any mask with bit 13 or above set, even if allowed bits are also set.

5. **Conditional preparation:** If classification is `19` and F-a2715f79cdcc returns `1`, F-352a899fb784 receives the owner and `1`. If the latter result is `2` or `0xFFFFFFFF`, that conditional call does not occur.

6. **Wait-cell reset:** If the wait dependency returns zero twice and nonzero on the third attempt, it is called three times. On each entry, both provided cells contain the seed captured after submission, regardless of writes made during earlier attempts.

7. **Matching owner:** If the owner field matches the entity after waiting, it is cleared before notification. If the record field remains `16`, F-fa1e7c5a0d15 receives `1`. If that field has validly become `11`, the clear still occurs but the notification does not.

8. **Status preservation:** If the table byte is `0xFE` when inspected, F-ca1c4e2f14bb receives false and the later direct write stores `0xFE`. For an inspected value of `0x7F`, it receives true and the later write stores zero.

## Uncertainties

The following remain **documented_only**:

- Semantic meanings of the classification, record fields, operation argument `8`, and final subsystem operations.
- Valid owner and entity domains, alignment requirements, table capacity, and safe indexed-arithmetic bounds.
- Record byte order and the three uninitialized record bytes.
- Dependency effects on globals, owner storage, the submitted record, resource ownership, and reentrancy.
- Whether the wait operation can terminate for every supported entity.
- Whether the non-returning invalid-mask operation terminates execution, transfers control elsewhere, or never completes.
- The context accessor's identity and guarantees.
- The concrete target and ownership contract of recipient dispatch.

## Evidence

**Established static evidence:** The supplied function body supports the validation mask, 32-bit argument usage, recipient order, record offsets and size, dependency argument relationships, wait condition, owner-field offset, table stride, status-byte rule, and final call order described above.

**Inferred intent:** The aggregate sequence resembles coordinated entity teardown or deactivation. This interpretation does not add dependency semantics or ownership guarantees.

**Evidence limits:** Only the supplied source representation and dependency map were available. Dependency implementations and runtime behavior were not inspected. No runtime tests have been performed.
