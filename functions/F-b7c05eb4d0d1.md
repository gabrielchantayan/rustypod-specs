<!-- Trusted provenance: F-b7c05eb4d0d1; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-b7c05eb4d0d1

## Purpose

F-b7c05eb4d0d1 conditionally populates a caller-provided record using values obtained from three dependencies. It writes an initial one-byte marker and, when that marker is nonzero, processes one-byte tags. Tags 0 and 1 select a four-byte field at byte offset 4; tag 2 selects a four-byte field at byte offset 8; tag 3 terminates processing. Other tags cause processing to continue without a four-byte field write for that iteration.

Four selector values bypass this processing entirely.

The established behavior is a sequence of dependency calls, record writes, and continuation checks. Interpreting the operation as decoding a tagged input stream is **inferred intent**, not an established dependency contract. In particular, the available evidence does not establish what the dependencies read, whether they consume input, or what their status checks mean.

The function returns a 32-bit value of either 0 or 1. A return of 1 does not necessarily mean that the record was populated: bypass selectors also return 1 without accessing the record.

## Interface

The function has four inputs:

| Input | Established representation | Role |
|---|---|---|
| Context | Opaque 32-bit value | Forwarded unchanged as the first argument of every dependency call. |
| Record | Byte-addressed pointer | Destination for the marker, current tag, and selected four-byte values. |
| Selector | 32-bit integer | Determines whether processing is bypassed. |
| Initial auxiliary value | Opaque 32-bit value | Initializes mutable call-local state used during non-bypassed processing. |

The return value is a 32-bit scalar:

- **0:** a call to F-8a872239bf2c returned zero.
- **1:** processing was bypassed, the initial marker was zero after the first successful continuation check, or tag processing terminated normally.

“Successful continuation check” means only that F-8a872239bf2c returned a nonzero value. No stronger meaning is established.

### Record layout

Offsets are measured in bytes from the supplied record pointer.

| Offset | Width | Established use |
|---|---:|---|
| 0 | 1 byte | Initial marker. |
| 1 | 1 byte | Most recently obtained tag. |
| 2–3 | 2 bytes | Not directly accessed by this function. |
| 4–7 | 4 bytes | Value written for tag 0 or tag 1. |
| 8–11 | 4 bytes | Value written for tag 2. |

These accesses do not establish a complete public structure definition. Padding, additional fields, and broader object ownership are unknown.

A caller seeking to support every ordinary processing path must provide writable storage through byte offset 11, readable storage for the marker and tag, and an address suitable for the target’s four-byte stores at offsets 4 and 8. A path that bypasses processing does not access the record. Other paths can access only a subset of the listed fields.

No byte order is established for the four-byte fields. They receive 32-bit dependency results through native four-byte stores rather than an explicitly specified byte serialization.

### Call-local state

Two four-byte local objects participate in dependency calls:

- A shared state value initialized to zero on every invocation.
- An auxiliary value initialized from the fourth input only on the non-bypassed path.

Dependencies receive the address of the shared state. F-2b01710262ca also receives the address of the auxiliary value. Their values are not reset between iterations.

These are call-local objects, not caller-owned output objects. The supplied body does not establish whether a dependency retains their addresses; retaining and subsequently using such addresses would require an external lifetime contract that is not available here.

## Behavior

### 1. Selector bypass

Selectors **2, 7, 4, and 9** all produce the same externally visible behavior:

- No dependency is called.
- The record is neither read nor written by this function.
- The function returns 1.

The shared local state is initialized to zero before this decision, but its address does not escape on a bypass path. The initial auxiliary input has no observable use on that path.

All other selector values enter the same processing path. There is no further distinction among them in the supplied body.

### 2. Initial marker

For a non-bypassed selector, the auxiliary local is initialized from the fourth input. The function then performs this ordered sequence:

1. Calls F-d54421325734 with the context, the value 1, and the address of the shared state.
2. Writes the one-byte result to record offset 0.
3. Calls F-8a872239bf2c with the context, the value 1, and the address of the same shared state.
4. Returns 0 immediately if that call returns zero.
5. Otherwise, reads the marker at record offset 0.

The marker write precedes the continuation check. A failed check therefore does not prevent or roll back that write.

If the marker read after the successful check is zero, processing ends with return value 1. Offset 1 and both four-byte fields are not directly written on this path.

If the marker is nonzero, tag processing begins. No particular nonzero marker value has distinct behavior.

The marker decision is based on the record contents read **after** F-8a872239bf2c returns, not on a separately retained copy of the earlier dependency result. If a dependency can modify the record through an alias, that modification can affect the decision.

### 3. Tag acquisition and continuation check

Each tag iteration begins with the following ordered actions:

1. Calls F-2b01710262ca with the context and addresses of both local state objects.
2. Stores the low eight bits of its result at record offset 1.
3. Calls F-8a872239bf2c with the context, the current 32-bit auxiliary value, and the address of the shared state.
4. Returns 0 immediately if that check returns zero.

The tag is written before the check. A failure at this point preserves that write and any earlier record writes; there is no rollback.

The auxiliary value passed by value to the check is the value present after the preceding F-2b01710262ca call. It need not equal the original fourth input.

After a nonzero check result, the function reads the tag from record offset 1 and selects the next action.

### 4. Tags 0 and 1

The comparison that selects this branch treats the stored tag as an **unsigned eight-bit value** and tests whether it is less than 2. Consequently, exactly byte values 0 and 1 enter this branch.

For either value:

1. Calls F-2b01710262ca with the context and addresses of the same two local state objects.
2. Writes its full 32-bit result at record offset 4.
3. Calls F-8a872239bf2c with the context, the current auxiliary value, and the address of the shared state.
4. Returns 0 if that check returns zero.

Tags 0 and 1 have identical behavior here. The supplied body does not establish different interpretations for their values.

Repeated occurrences overwrite the same four-byte field. Earlier values are not accumulated by this function.

### 5. Tag 2

For tag byte 2, the sequence is the same as for tags 0 and 1 except that the full 32-bit result is stored at record offset 8.

A failure of the following continuation check returns 0 after the field has already been written.

Repeated tag-2 occurrences overwrite the same field.

### 6. Tag 3 and other tags

A tag other than 0, 1, or 2 causes no four-byte value acquisition and no four-byte field write in that iteration.

At the end of every iteration that has not already returned 0, the function reads the tag at record offset 1:

- If it is 3, the loop ends and the function returns 1.
- Otherwise, another tag iteration begins.

Thus, an ordinary tag-3 iteration still performs tag acquisition, writes offset 1, and performs its continuation check before terminating. A zero result from that check returns 0 rather than 1.

Byte values 4 through 255 are not rejected explicitly. They continue the loop without a payload-field operation. In particular, values 128 through 255 do not enter the unsigned “less than 2” branch, regardless of the target’s plain-character signedness.

The end-of-iteration decision is a new record read. After a payload operation and its check, it is not necessarily the same value that selected the payload branch if a dependency has modified the record through an alias.

### 7. Progress and termination

There is no explicit iteration limit, byte count limit, timeout, or independent end-of-input check in this function.

Normal loop termination requires a final tag read equal to 3. Otherwise, processing continues until a continuation check returns zero, a dependency fails to return, or an externally governed execution failure occurs.

## Side effects

Direct caller-visible writes are limited to the record locations described above. They occur incrementally:

- Marker before its check.
- Tag before its check.
- Selected four-byte value before its check.

No direct record initialization or clearing occurs. Fields not reached on a particular path retain their prior contents unless a dependency modifies them indirectly. A return of either 0 or 1 can therefore leave a partially populated record.

The function performs no explicit allocation, deallocation, ownership transfer, or resource release. This does not establish that its dependencies perform none.

The context is forwarded unchanged, but the function does not establish preservation of any resource, object, or global state reachable through it. Dependencies may have additional side effects not visible in the supplied body.

There is no explicit null-pointer check, destination-size check, alignment check, or exception handling. The ordinary processing path requires valid record access and valid dependency-specific inputs. No defined return code is established for invalid memory, inaccessible context state, dependency exceptions, or other execution faults.

## Dependencies

| Opaque ID | Use in this function | Established result handling |
|---|---|---|
| F-d54421325734 | Initial marker acquisition; receives context, constant 1, and shared-state address. | One-byte result is written to offset 0. |
| F-2b01710262ca | Tag and four-byte value acquisition; receives context and addresses of both mutable locals. | Low eight bits are used for tags; all 32 bits are used for four-byte field writes. |
| F-8a872239bf2c | Continuation gate after each acquisition and corresponding record write. | Zero causes immediate return 0; every nonzero result permits further processing. |

The shared state begins at zero, but the function does not directly inspect its subsequent contents. Any error-state meaning is **inferred intent**.

The auxiliary local is passed by address to F-2b01710262ca and by current value to later calls of F-8a872239bf2c. Treating it as a length, count, encoding parameter, or consumed-byte total would be speculative.

No dependency contract for global preservation, record aliasing, memory ownership, or input advancement is supplied.

## Acceptance examples

These are **conditional static acceptance examples**, not executed tests. Except where stated otherwise, assume dependencies return normally, record accesses are valid, and dependencies do not indirectly change the record.

1. **Bypass selector.** With selector 7, the function returns 1 without calling any dependency or accessing the record. The same applies to selectors 2, 4, and 9.

2. **Initial check fails.** For a non-bypassed selector, marker acquisition yields byte 5 and the following check returns zero. Offset 0 is written with 5, then the function returns 0. No tag acquisition occurs.

3. **Zero marker.** Marker acquisition yields zero and its check returns nonzero. Offset 0 becomes zero and the function returns 1. Existing tag and payload fields remain directly untouched.

4. **Immediate terminating tag.** The marker is nonzero, its check succeeds, and the first tag acquisition yields a result whose low byte is 3. After storing 3 at offset 1 and obtaining a nonzero tag check, the function returns 1 without acquiring a four-byte value.

5. **Tag-result truncation.** A tag acquisition result of hexadecimal 00000102 produces stored tag 2. It selects a full-width value acquisition for offset 8, provided the tag check succeeds. Upper result bits do not participate in tag classification.

6. **Both payload fields.** With successful checks throughout, an iteration for tag 0 writes its acquired 32-bit value at offset 4; a later tag-2 iteration writes its acquired value at offset 8; a subsequent tag 3 terminates processing. Offset 1 ends as 3.

7. **Payload check fails.** Tag 1 is accepted, its acquired value is stored at offset 4, and the following check returns zero. The function returns 0 with that value already stored.

8. **Unrecognized tag continues.** After a successful check for byte 255, neither four-byte field is directly written during that iteration. Another tag acquisition follows.

9. **Auxiliary-state change.** If F-2b01710262ca changes the auxiliary local from its initial value to another 32-bit value, the immediately following check receives the changed value.

10. **Aliased marker change.** If the first check returns nonzero but changes the marker to zero through an alias, the function returns 1 without entering the tag loop, even if the original marker acquisition produced a nonzero byte.

## Uncertainties

The following remain **documented_only**:

- Exact dependency semantics, including input consumption and failure causes.
- Whether the context represents a pointer, handle, stream, or another object.
- Meaning of the marker, tags, shared state, auxiliary value, and payload fields.
- Target endianness and exact alignment requirements.
- Whether dependencies can alias the record or alter global state.
- Whether dependencies retain pointers to call-local storage.
- Behavior on invalid pointers, storage exhaustion, faults, or nonreturning dependencies.
- Any external invariant guaranteeing eventual tag 3 or a failing check.

The function contains no explicit arithmetic beyond fixed byte-offset addressing, and no increment or decrement of either local state value. Any arithmetic performed by dependencies is outside this evidence.

## Evidence

The supplied static function body establishes the selector bypass, scalar widths, local initialization, dependency-call ordering, record access offsets, unsigned tag comparison, zero/nonzero gates, loop termination condition, and absence of rollback.

The private dependency map supplies the opaque dependency identities used above. It does not supply their implementations or behavioral guarantees.

No runtime tests have been performed. The acceptance examples describe consequences of the visible control flow under their stated assumptions; they are not observations of execution.
