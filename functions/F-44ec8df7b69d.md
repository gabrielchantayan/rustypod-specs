<!-- Trusted provenance: F-44ec8df7b69d; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-44ec8df7b69d

## Purpose

F-44ec8df7b69d processes mutable buffered text using classification data and substantial implicit parsing state. It acquires and assembles records, distinguishes bracket-delimited constructs from assignment-shaped constructs, inserts marker bytes into text, creates an assignment-related object, and invokes downstream processing.

**Established static behavior:** a distinguished zero-length acquisition returns the 32-bit value 1. Failures reaching shared reporting return the 32-bit value 0. Successful construct processing resumes acquisition rather than returning immediately.

**Inferred intent:** the syntax resembles configuration input with sections, qualified names, assignments, and continued records. These terms describe apparent roles, not established dependency semantics.

**Documented_only:** the complete calling convention, classification meanings, marker value, dependency contracts, and ownership model are unavailable.

## Interface

### Explicit inputs and result

| Item | Contract |
|---|---|
| Text position | Pointer to mutable byte storage, initially retained as a possible name or qualifier start. It is replaced after completed record assembly. |
| Current symbol | Unsigned 32-bit value. Its initial classification lookup uses the full value, without narrowing it to a byte. |
| Result | 32-bit value 1 on the distinguished acquisition exit, or 0 after shared failure reporting, provided invoked dependencies return normally. |

The function does not verify consistency between the initial symbol and the pointed-to byte. It does not establish pointer validity or classification-table bounds.

### Implicit entry state

The visible behavior additionally requires:

- A classification context with a 32-bit table pointer at byte offset 4 and a 32-bit state field at offset 8.
- A buffer-related object with a 32-bit data pointer at offset 4.
- A signed 32-bit accumulated offset.
- A 32-bit pending-object reference.
- An unsigned 32-bit continuation indicator.
- A 32-bit marker value.
- A saved comparison value for the classification context’s field at offset 8.
- An optional current-context pointer.
- A signed 32-bit record counter.
- An opaque input handle.
- An optional destination for a 32-bit counter write.

This is a description of required live state, not a complete public ABI.

### Widths and preconditions

Classification entries are unsigned 16-bit values. Their addresses use a symbol index scaled by two. Symbols obtained from text are unsigned bytes widened for indexing; the explicit initial symbol is not similarly constrained.

Byte writes use the marker’s low eight bits. Object-field and context-field marker writes use all 32 bits. Pointer-bearing fields and the visible arithmetic use a 32-bit representation.

Safe execution requires accessible classification entries, valid returned pointers, writable marker destinations, and representable offsets and sizes. No capacity, alignment, address-order, or arithmetic-overflow checks are visible. In particular, an acquisition destination must permit a write at relative offset 511. A buffer reference that is tested for nonzero during cleanup is nevertheless dereferenced earlier on a successful acquisition path; that later test does not make a null buffer valid there.

## Behavior

### Dispatch and acquisition

At each classification dispatch, the function reads the current symbol’s classification entry and captures the marker’s low byte. Bit `0x08` selects acquisition. Otherwise, the symbol determines construct processing.

Each acquisition attempt has this observable order:

1. Call F-aba38163eef3.
2. Read a value from a static storage location.
3. Test the gate result.
4. If the result is zero, enter the general error path.
5. Otherwise, load the buffer data pointer and add the accumulated offset.
6. Write the captured marker byte at that append destination.
7. Call F-5e31366444c1 with the input handle, append destination, and previously captured static value.
8. Write the marker byte at append destination plus 511.
9. Call F-1cb585febd84 on the append destination and interpret its result as a signed 32-bit length.

The static-value read occurs even when the gate returns zero. On success, its value is captured before the destination marker write, rather than read at the later call. The result of F-5e31366444c1 is not visibly tested.

The marker byte captured at classification dispatch remains in use when processing branches directly back to acquisition. Such a branch does not itself perform another classification dispatch.

If the measured length and incoming continuation indicator are both zero, the function conditionally calls F-28097aea1ffc for a nonzero buffer reference, then F-5990a26eaaa3 for a non-null current-context pointer, and returns 1.

This exit does not publish the record counter, issue the shared diagnostic, inspect the pending object, or restore the classification context’s field at offset 8.

### Record assembly

A zero measurement with nonzero incoming continuation does not take the distinguished exit.

For measurements continuing into assembly, the continuation indicator is first cleared. A positive measured region is shortened while its final byte is carriage return (`0x0D`) or line feed (`0x0A`).

If the original measurement was zero, or trimming removed at least one byte:

- Write the marker byte at the resulting endpoint.
- Increment the record counter once.

Otherwise, set continuation to 1. Consequently, a positive chunk without a trailing line ending requests another acquisition, while removing multiple line-ending bytes still increments the counter only once.

The resulting length is added to the accumulated offset. The pending-object reference is then cleared, without an accompanying cleanup call.

When the accumulated offset is positive, the function examines the assembled suffix using the buffer object’s then-current data pointer. If the final byte has classification bit `0x20`, and either it is the only accumulated byte or the preceding byte lacks that bit, the offset decreases by one, continuation becomes 1, and acquisition resumes.

This rule examines only the final two bytes; it is not an odd/even count of consecutive continuation characters. The decrement itself does not erase the removed byte. A subsequent acquisition writes its marker at the new append position.

If continuation remains nonzero for any reason, acquisition repeats. Otherwise, the function resets the accumulated offset to zero, calls F-5c60aa004548, obtains a text position from F-c7605d5fc33e, reads its first byte as the next symbol, and restarts classification.

Negative measurements are not explicitly rejected. Their arithmetic and memory-safety consequences are not established as valid behavior.

### Bracket-delimited constructs

For symbol `0x5B`:

- Call F-c7605d5fc33e once, without using its returned position in a visible read.
- Repeatedly call F-697dcb2944b2 and then F-c7605d5fc33e.
- Inspect the first byte at the latter returned position.

Byte `0x5D` ends the scan. A zero byte encountered first selects the general error path. Every other byte causes another iteration.

After finding the closing byte, the function writes the marker through the position returned by F-697dcb2944b2 in that iteration. That writable position is not proven identical to the position containing the closing byte.

Next, F-c0e06c3e7b96 is called. A zero result enters shared reporting directly, bypassing the general error notification. Otherwise, F-ee2fa3f97b75 is evaluated. Its nonzero result resumes acquisition. If it returns zero, F-a2dadb513b63 is evaluated; nonzero resumes acquisition, and zero selects the general error path.

No explicit nesting or escape rule is visible. Unknown position-provider behavior prevents stronger claims about the accepted grammar.

### Assignment recognition and marker placement

For every non-bracket construct, F-697dcb2944b2 supplies a candidate delimiter position. Its first byte is read. Its second byte is read only if the first is a colon.

If both bytes are colons:

- Replace the first colon with the marker byte.
- Retain the original text position as the qualifier candidate.
- Use the position two bytes after the delimiter as the name start.
- Call F-697dcb2944b2 again for the assignment delimiter position.

Otherwise, retain the original text position as the name start and retain no explicit qualifier.

F-c7605d5fc33e then supplies a position whose first byte must be `0x3D`. Any other byte selects the general error path.

On recognition:

1. Write the marker through the retained assignment delimiter position.
2. Call F-c7605d5fc33e again to obtain a value-related boundary.
3. Starting one byte after the previously returned equals-sign position, scan forward to the first byte whose classification includes `0x08`.
4. Inspect preceding positions backward, skipping bytes with classification bit `0x10`, but stop on reaching the value-related boundary without classifying that boundary byte.
5. Write the marker immediately after the position where this backward examination stops.

The forward scan always advances beyond the equals-sign position before its first classification test. It has no explicit buffer-end check. A zero byte terminates it only if that byte’s classification includes `0x08`.

The backward examination also makes at least one backward step. Its safety depends on the relationship between the endpoint and the returned boundary; their order is not validated. Marker insertion does not establish string termination or name/value isolation unless additional marker and dependency contracts supply those properties.

### Assignment object and downstream processing

F-43da6b23bd44 is called with size argument 12 and two zero arguments. Its result becomes the pending-object reference. Zero selects the general error path.

For a nonzero result, the visible object layout is:

| Byte range | Visible use |
|---|---|
| 0–3 | No explicit initialization by this function. |
| 4–7 | Storage for the subsequent allocation-shaped result; later reloaded. |
| 8–11 | Full 32-bit marker value. |

If no explicit qualifier was retained, the current-context pointer becomes the qualifier candidate.

The function measures the name with F-1cb585febd84, adds one to that signed result, and passes the resulting size with two zero arguments to F-43da6b23bd44. It stores the returned reference at object offset 4, then stores the full marker at offset 8. The allocation result is tested for zero; zero selects the general error path.

On a nonzero allocation result, the name is measured again. **After this second measurement, the function reloads the pending object’s field at offset 4.** F-c73847cd39ec receives that reloaded value as its destination, the name start as its source, and the second measurement plus one as its size.

The two measurements are distinct calls. Their results need not agree. The second call may also affect accessible object storage: the destination supplied to F-c73847cd39ec is not necessarily the earlier allocation result. The reloaded destination is not checked again, even if it has become zero. Neither allocation nor copying semantics are established by call shape alone.

F-c0e06c3e7b96 follows. Zero enters shared reporting directly. Otherwise, an unidentified comparison dependency receives the qualifier candidate and current-context pointer.

Eligibility for F-8b7078ed55dc is established by any of:

- A zero comparison result.
- A nonzero comparison result followed by nonzero F-ee2fa3f97b75.
- A nonzero comparison result, zero F-ee2fa3f97b75, and nonzero F-a2dadb513b63.

These alternatives are short-circuited. Failure of all applicable alternatives selects the general error path.

F-8b7078ed55dc is called only when eligible. Its zero result selects the general error path. Its nonzero result clears the pending-object reference and resumes acquisition. Clearing the reference suppresses later cleanup through that reference; it does not prove ownership transfer.

## Side effects

### Failure and reporting order

The general error path first calls F-4fc89ec41a6d with `0x0E` and `0x66`. A zero result from F-c0e06c3e7b96 skips this notification and enters the following shared sequence directly:

1. Conditionally call F-28097aea1ffc for a nonzero buffer reference.
2. Conditionally call F-5990a26eaaa3 for a non-null current-context pointer.
3. If the counter destination is non-null, write the current 32-bit record counter there.
4. Call F-ff84c77c15bc with temporary storage, size argument 13, an opaque formatting argument, and the counter.
5. Call F-968c88301966 with category 2, a line-related label, and the temporary storage address.
6. Examine the classification context’s field at offset 8. If it differs from the saved comparison value and is nonzero, call F-fcaed3dadcdf, then overwrite the field with the full marker.
7. If the pending object is nonzero, perform its conditional field cleanup and final cleanup.
8. Return 0.

The context field is read for comparison against the saved value and, only if unequal, read again for the zero comparison. No stable snapshot across these reads is established.

Pending-object cleanup first reads offset 4 and conditionally calls F-5990a26eaaa3. It then reads offset 8 and conditionally calls the same dependency again. Finally, it calls F-5990a26eaaa3 with the pending-object reference explicitly.

The first two field-associated cleanup calls have no explicit argument in the supplied representation. Their actual targets remain unresolved. The offset-8 read occurs after the preceding conditional call; field preservation or object lifetime across that call must not be assumed.

### Mutation and ownership limits

Text writes occur before some subsequent failures and are not rolled back. Dependencies can observe already inserted markers.

The marker is not established as zero. Its full-width value can also make object offset 8 nonzero immediately after initialization.

Allocation-shaped and cleanup-shaped calls suggest resource management, but do not establish ownership, successful copying, release semantics, exactly-once cleanup, or preservation of globals and accessible memory. Safety requirements across dependency calls remain preconditions, not inferred guarantees.

## Dependencies

| Opaque ID | Observed role |
|---|---|
| F-aba38163eef3 | Acquisition gate, followed unconditionally by the static-value read. |
| F-5e31366444c1 | Receives input handle, append destination, and captured static value. |
| F-1cb585febd84 | Measurement-shaped operation used for assembly and name sizes. |
| F-28097aea1ffc | Conditional buffer-related cleanup-shaped operation. |
| F-5990a26eaaa3 | Cleanup-shaped operation with explicit or unresolved arguments. |
| F-5c60aa004548 | Called after completed assembly. |
| F-c7605d5fc33e | Supplies parsing positions. |
| F-697dcb2944b2 | Supplies writable delimiter-related positions. |
| F-c0e06c3e7b96 | Gate whose zero result bypasses general error notification. |
| F-ee2fa3f97b75 | First short-circuited processing alternative. |
| F-a2dadb513b63 | Second processing alternative. |
| F-43da6b23bd44 | Allocation-shaped operation for object and name-related storage. |
| F-c73847cd39ec | Receives reloaded destination, name source, and second measured size plus one. |
| F-8b7078ed55dc | Final assignment-processing predicate. |
| F-4fc89ec41a6d | General error notification. |
| F-ff84c77c15bc | Counter-formatting-shaped operation. |
| F-968c88301966 | Diagnostic emission-shaped operation. |
| F-fcaed3dadcdf | Context-related operation before marker replacement. |

One comparison dependency has no supplied opaque ID. Its identity and semantics are documented_only; its two pointer arguments and zero/nonzero branching role are established.

## Acceptance examples

These are conditional static acceptance cases, not executed tests. They assume valid memory, representable arithmetic, and normally returning dependencies.

1. **Failed acquisition gate:** when F-aba38163eef3 returns zero, the static argument value is still read before the failure branch. No append marker or F-5e31366444c1 call occurs on that attempt.

2. **Captured acquisition argument:** if the append marker write changes storage backing the static argument through aliasing, F-5e31366444c1 still receives the value captured before that write.

3. **Distinguished exit:** a zero measurement with zero incoming continuation causes conditional buffer/context cleanup and return 1, without counter publication or shared reporting.

4. **Zero measurement during continuation:** a zero measurement with nonzero incoming continuation enters assembly, writes an endpoint marker, and increments the counter once. Any positive preexisting assembled suffix is still subject to the continuation-suffix rule.

5. **Line endings and continuation:** a chunk ending in carriage return followed by line feed loses both from its effective length but increments the counter once. A suffix whose last two bytes both have bit `0x20` does not trigger the suffix-removal rule.

6. **Accepted bracket construct:** reaching a closing bracket, passing F-c0e06c3e7b96, and obtaining nonzero from F-ee2fa3f97b75 causes the associated marker write and resumed acquisition. F-a2dadb513b63 is skipped. Reaching zero before a closing bracket instead selects general error reporting.

7. **Qualified assignment:** recognized adjacent colons, a recognized equals sign, valid scan boundaries, nonzero allocation-shaped results, and successful downstream predicates produce the specified marker writes, allocation-shaped calls, and F-c73847cd39ec invocation. Success clears the pending reference and resumes acquisition. This case does not establish string isolation or copied name bytes.

8. **Changed destination field:** suppose the first name-related allocation returns nonzero, but the second measurement changes object offset 4. F-c73847cd39ec receives the changed field value, including zero if that is its new value, without another destination check.

9. **Allocation failures:** zero from the 12-byte allocation selects general error reporting with a zero pending reference. Zero from the name-related allocation reaches the same path after offset 4 has been written as zero and offset 8 has received the full marker.

10. **Direct reporting gate:** zero from F-c0e06c3e7b96 after either construct type enters shared reporting without an intervening F-4fc89ec41a6d call.

## Uncertainties

The following remain documented_only:

- Complete ABI, hidden state origins, classification-table extent, and semantic meanings of classification bits.
- Marker value and whether any marker write acts as a string terminator.
- Contents and mutability of static argument storage and diagnostic formatting data.
- Measurement, allocation, copying, comparison, processing, and cleanup semantics.
- Effects of dependencies on globals, object fields, text, and object lifetimes.
- Safety of null qualifier/current-context arguments to the unidentified comparison dependency.
- Behavior at negative measurements, signed counter or offset overflow, length-plus-one overflow, scaled-index overflow, or address wraparound.
- Behavior when dependencies fail without returning zero, do not return, or invalidate storage needed later.

No missing dependency contract is replaced with a standard-library assumption.

## Evidence

This contract uses only the supplied static function representation and opaque dependency mapping. Established claims concern visible access widths, offsets, branch conditions, call arguments, short-circuiting, mutation order, and return paths.

The acquisition description includes the unconditional static-value read after the gate call. The assignment description distinguishes the checked allocation result from the destination field reloaded after the second measurement. Neither marker insertion nor the copy-shaped call is treated as proof of isolated or copied strings.

Parser terminology and resource-management roles are inferred intent. Unavailable dependency behavior remains explicitly unresolved.

No runtime tests have been performed.
