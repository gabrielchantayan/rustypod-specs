<!-- Trusted provenance: F-eddc89ee2a0a; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-eddc89ee2a0a

## Purpose

F-eddc89ee2a0a combines two ordered activities:

1. Conditionally refresh a one-byte state in a caller-supplied object and notify another component when a previously nonzero state changes.
2. Process a separately obtained shared record, returning either a dependency result or a locally selected status.

These activities are not transactional. Object-state writes and notifications can occur before the function discovers that no shared record is available, encounters an ownership mismatch, or returns early through a dependency.

The supplied static evidence establishes the reads, writes, branches, and calls below. It does not establish the application meaning of the state values, the shared record’s type marker, or the numerical return statuses.

## Interface

### Inputs

The function has three machine-integer arguments, represented as 32-bit values in the supplied evidence:

| Argument | Interpretation established by the function |
|---|---|
| Object address | Base address used for fixed-offset memory accesses. |
| Refresh control | Zero permits reuse of an existing nonzero state; any nonzero value forces the state-refresh branch. |
| Auxiliary value | Has no direct influence on the observed branch decisions or dependency arguments. Whenever its value would matter, it has first been replaced by an object field. |

The object address is used as an address, not merely an integer identifier. There is no explicit null check, bounds check, or memory-access error handling.

The auxiliary argument is therefore behaviorally irrelevant to the direct logic shown. This does not imply anything about an ABI beyond the supplied function body.

### Object layout

The following offsets are relative to the supplied object address:

| Offset | Width | Access and role |
|---|---:|---|
| `0x08` | 32 bits | Read to form the notification target address. |
| `0x0C` | 32 bits | Read as the first argument to F-18229149dff0. |
| `0x1AE` | 8 bits | Read as the previous state; conditionally written as provisional and final state. |
| `0x1B2` | 8 bits | Tested for zero versus nonzero to enable state computation. |
| `0x1B8` | 32 bits | Read only when the enable byte is nonzero; its value is tested for zero. |

The state byte is initially interpreted as an unsigned value from 0 through 255. The enable byte is tested only for zero, so its signed interpretation does not affect that decision.

### Shared record layout

A shared pointer is obtained from a global pointer slot. When nonnull and when execution reaches the relevant accesses, the pointed-to record has these observed fields:

| Offset | Width | Access and role |
|---|---:|---|
| `0x00` | 32 bits | Compared with a global marker. |
| `0x04` | 32 bits | Compared with the result of F-196812b4b122; may later be cleared. |
| `0x0E` | 16 bits | Read as an unsigned control field. |
| `0x12` | 16 bits | Conditionally decremented and written; may be explicitly cleared. |
| `0x14` | Address within record | Passed to F-259d2fe281b0. |

These accesses do not establish a complete record layout, allocation size, ownership model, or byte order.

### Output

The function returns a 32-bit integer result:

- `26` when the captured shared-record pointer is null.
- A nonzero result from F-621dae46a146 when that dependency is conditionally called and returns nonzero.
- `5` when the shared record’s ownership-like field does not match the result of F-196812b4b122.
- `0` when matching ownership and the control/count condition defer the final dependency call.
- The result of F-259d2fe281b0 when that dependency is reached.

The meanings of `0`, `5`, and `26` are not established. They must not be labeled success, permission failure, or missing-resource failure without additional evidence.

## Behavior

### Entry ordering and captured state

The shared-record pointer is loaded from its global slot before an unconditional entry hook is called. The supplied dependency map does not assign an opaque function ID to this hook.

After the hook returns, the object’s state byte at `0x1AE` is read and retained as the previous-state value.

This order matters:

- A hook change to the object’s state can affect the previous-state value.
- A hook change to the global shared-pointer slot does not replace the pointer already captured by this invocation.
- The function provides no evidence that the hook preserves either global state or pointed-to memory.

All subsequent shared-record processing uses the captured pointer. The global pointer slot is not reread.

### Conditional object-state refresh

State refresh occurs when either:

- The retained previous-state byte is zero; or
- Refresh control is nonzero.

Otherwise, this function performs no state-refresh writes and makes neither of the state-related dependency calls described below. It still proceeds to shared-record processing.

When refresh occurs, the candidate result initially equals the 32-bit value `1`.

The enable byte at object offset `0x1B2` is then read. If it is nonzero, the 32-bit field at `0x1B8` is read, replacing the incoming auxiliary argument for the subsequent zero test.

F-18229149dff0 is called only when both of these conditions hold:

- The enable byte is nonzero.
- The field at `0x1B8` is nonzero.

Immediately before that call, the function writes the byte value `6` to object offset `0x1AE`. The dependency can therefore observe this provisional state.

F-18229149dff0 receives:

1. The 32-bit value read from object offset `0x0C`.
2. The object address itself.

Its result becomes the candidate result. If the call is not made, the candidate remains `1`.

Afterward, the low eight bits of the candidate result are written to object offset `0x1AE`. This write occurs even when the candidate is the default value `1`, and even when the stored byte would equal the byte already present.

### Change notification

After the final state-byte write, notification occurs exactly when both conditions hold:

- The retained previous-state byte is nonzero.
- The full 32-bit candidate result differs from that zero-extended previous-state byte.

The comparison is not between two eight-bit stored states. A candidate result with nonzero upper bits can trigger notification even when its low byte matches the previous-state byte.

F-ebd0ee53699f receives:

1. The address formed from the 32-bit value at object offset `0x08`, advanced by `0xBAC` bytes.
2. A value supplied from a separate global location.
3. Zero.

The object field used to form the first argument is read after the state-computation call and final state write. Thus, a change to that field by F-18229149dff0 can affect the notification target.

The notification return value is not used. Any other effects of the notification dependency remain possible; the caller does not restore state afterward.

### Shared-record presence and optional dispatch

Only after the state-refresh phase is complete does the function act on the captured shared pointer.

If that pointer is null, the function returns `26`.

Otherwise, it reads the record’s 32-bit marker at offset `0x00` and compares it with a global marker value. When the values match, F-621dae46a146 is called with:

1. The captured record pointer.
2. Zero.

A nonzero result from this dependency is returned immediately. In that case, the later control-field read, ownership comparison, count handling, and F-259d2fe281b0 call do not occur in this function.

A zero result permits processing to continue. There is no assumption that this dependency leaves the record or globals unchanged.

If the marker comparison does not match, the dependency is skipped and processing continues.

### Ownership comparison and count handling

After the optional dispatch returns zero, or after it is skipped, the function reads the unsigned 16-bit control field at record offset `0x0E`.

It then calls F-196812b4b122 without explicit arguments. Only after that call returns does it read the 32-bit field at record offset `0x04` for comparison with the dependency result.

Consequently:

- The control field used for the later decision is a snapshot taken before this call.
- The ownership-like field is read after this call.
- Changes by this dependency to those two fields can therefore affect the invocation differently.

If the ownership-like field does not match the dependency result, the function returns `5`. This branch performs no count or owner-field writes itself.

If ownership matches, bits 4 and 5 of the captured control field determine whether count handling is required. Equivalently, the relevant class is the control field masked by `0x30` and shifted right by four bits. Other control bits do not affect this decision.

When the class is not `2`, the function proceeds directly to record clearing and the final dependency call.

When the class is `2`, the function reads the 16-bit count at offset `0x12`, subtracts one, and writes the resulting 16-bit value back. Continuation depends on whether that stored-width result is zero:

- A zero result proceeds to record clearing and the final dependency call.
- Any nonzero result returns `0`, leaving the decremented count in place and making no direct write to the ownership-like field.

This is a 16-bit decrement, not a saturating decrement. In particular, an initial count of zero produces the bit pattern `0xFFFF`, which is nonzero and therefore defers the final call.

### Record clearing and final dependency

When record clearing is reached, the function performs these direct writes in order:

1. Write zero to the 16-bit count at offset `0x12`.
2. Write zero to the 32-bit ownership-like field at offset `0x04`.

It then calls F-259d2fe281b0 with the address at record offset `0x14` and returns that dependency’s result.

For class `2` with an initial count of one, the decrement first writes zero, and the clearing step then writes zero to the same count field again. These are two distinct writes in the supplied ordering.

For other classes, the existing count is not decremented or tested before being cleared.

## Side effects

Direct writes are limited to the observed object state byte, record count, and record ownership-like field. Dependencies may produce additional effects that are not described by this body.

Important visibility and ownership constraints:

- There is no rollback of object-state changes when later record processing returns `26`, `5`, or a dependency result.
- The provisional state `6` is visible during F-18229149dff0.
- The final low-byte state write precedes change notification.
- Record clearing precedes F-259d2fe281b0 and is not conditioned on that dependency reporting any particular result.
- No direct allocation or deallocation is shown.
- Passing an interior record address to F-259d2fe281b0 does not establish whether that dependency releases, retains, or merely operates on memory.
- The captured record must remain valid for all accesses actually reached. The function contains no protection against a dependency invalidating it.
- No synchronization, atomicity, or reentrancy guarantee is established.

Address arithmetic is represented using 32-bit values in the supplied evidence. Valid execution requires the accessed addresses and derived addresses to be meaningful for the target environment; no overflow or address-validity checks are shown.

## Dependencies

| Opaque ID | Established interaction |
|---|---|
| F-18229149dff0 | Conditionally computes the candidate state from an object-associated 32-bit value and the object address. Its full result drives notification comparison; its low byte is stored. |
| F-ebd0ee53699f | Conditionally notified after the final state write, using a derived target address, a global-supplied value, and zero. Return value unused. |
| F-621dae46a146 | Called only for a matching record marker. A nonzero result terminates this invocation; zero continues processing. |
| F-196812b4b122 | Called before the ownership-like comparison. Its result is compared with the record field read after the call. |
| F-259d2fe281b0 | Called after clearing count and ownership-like fields. Receives the record’s interior address at `0x14`; its result is returned. |

An additional unconditional entry hook is present, but its opaque ID and contract are unavailable. Its return value is unused.

None of these interactions establishes that a dependency preserves globals, preserves object fields, or owns or releases passed memory.

## Acceptance examples

These are conditional predictions from static evidence, not executed tests. Unless otherwise stated, assume valid memory and dependencies that return normally without unmentioned mutations.

1. **Existing state reused.**  
   With previous state `7` and refresh control zero, the function leaves the state byte untouched and skips state computation and notification. Shared-record processing still occurs.

2. **Default initialization without notification.**  
   With previous state zero and the enable byte zero, the function writes state `1`. It does not notify, because the retained previous state is zero. If the captured shared pointer is null, the eventual return is `26`.

3. **Enabled flag with zero associated field.**  
   With refresh forced, a nonzero enable byte, and zero at object offset `0x1B8`, F-18229149dff0 is skipped. The candidate remains `1`, regardless of the incoming auxiliary argument.

4. **Provisional state observed by computation.**  
   With refresh enabled and a nonzero associated field, F-18229149dff0 is entered after the state byte has been written as `6`. If it returns `9`, the caller subsequently writes state byte `9`.

5. **Full-width change despite identical stored byte.**  
   With previous state `1`, forced refresh, and a candidate result of `257`, the final stored byte is `1`, but notification occurs because `257` differs from `1`.

6. **Early record dispatch result.**  
   If the marker matches and F-621dae46a146 returns `12`, the function returns `12` without performing the later ownership comparison or record-count handling. Earlier object-state effects remain.

7. **Ownership mismatch.**  
   If processing reaches the ownership comparison and the values differ, the function returns `5`. It does not directly clear or decrement the record count.

8. **Deferred class-2 record.**  
   With matching ownership, control class `2`, and initial count `2`, the count becomes `1`; the function returns `0` without calling F-259d2fe281b0.

9. **Class-2 transition to clearing.**  
   With matching ownership, control class `2`, and initial count `1`, the decrement writes zero. The function then explicitly clears the count again, clears ownership, and returns F-259d2fe281b0’s result.

10. **Zero count does not clear immediately.**  
    With matching ownership, control class `2`, and initial count zero, the count becomes `0xFFFF`. The function returns `0` without the final dependency call.

11. **Other control classes bypass decrement.**  
    With matching ownership and any control class other than `2`, the function clears count and ownership regardless of the previous count, then calls F-259d2fe281b0.

## Uncertainties

The following are **documented_only**:

- The application-level purpose of the object, record, markers, state values, and return statuses.
- The entry hook’s opaque identity and all dependencies’ internal behavior.
- Whether globals are mutable concurrently or through callbacks.
- Whether any dependency retains or releases either passed address.
- Allocation boundaries, alignment guarantees, byte order, and the complete ABI.
- Whether the count is conceptually a reference count, nesting depth, or another quantity.
- The meaning and representation of the global-supplied notification value.
- Behavior after invalid memory access, dependency nonreturn, or memory invalidation.

An inferred intent is that the record tracks an owner and a deferred-release count. That interpretation is not required by the established behavior and is not an ownership guarantee.

## Evidence

This contract is based solely on the supplied static function representation and dependency-ID mapping. No runtime tests have been performed.

Established evidence includes the explicit access widths and offsets, captured-pointer ordering, conditional calls, low-byte state storage, full-width notification comparison, 16-bit count update, ordered clearing writes, and return-value propagation.

Semantic labels such as “state,” “ownership-like,” “count,” and “notification” describe observed roles for readability. They do not establish undocumented application semantics or dependency contracts.
