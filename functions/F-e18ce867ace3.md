<!-- Trusted provenance: F-e18ce867ace3; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-e18ce867ace3

## Purpose

F-e18ce867ace3 measures an existing destination prefix or appends source characters to that prefix, depending on the arguments. Its result counts nonzero characters in the destination prefix, including any appended characters, but excludes the terminating zero.

The capacity argument is **not a universal access bound**. It limits the combined existing and appended prefix only when the source and destination addresses are both non-null and capacity exceeds one. Other calls with a non-null destination measure through its first zero regardless of capacity.

Even bounded operation does not guarantee destination termination. If the initial destination search consumes the allowance without finding zero, the function returns without modifying the destination.

**Inferred intent:** The behavior resembles bounded concatenation with a measurement fallback. This interpretation does not imply conventional library semantics, unconditional termination, or safe handling of arbitrary overlap.

## Interface

The function accepts three arguments in this order:

| Input | Meaning |
|---|---|
| Source address | Nullable address of character storage used when appending is enabled. |
| Destination address | Nullable address of character storage to measure or modify. |
| Capacity | Unsigned integer selecting the operating mode and, in bounded operation, limiting the combined nonzero prefix. |

The result is a signed integer. No separate status result or designated error value is established.

### Representation

Characters occupy consecutive character-sized addressable units. Zero terminates a sequence; every nonzero unit contributes one to its length. No encoding interpretation, multibyte decoding, normalization, or case conversion is visible.

The supplied representation establishes a signed integer counter and result, an unsigned integer capacity, and character-addressed storage. It does **not** independently establish their bit widths. In particular:

- Eight-bit characters are not established.
- A 32-bit result or capacity is not established.
- Pointer width and calling convention are not established.
- Character signedness is not established.

Nonzero characters are transferred as character values without an explicit transformation. Character signedness does not change the zero/nonzero stopping distinction.

### Storage and ownership

The caller supplies and retains the storage. No allocation, release, ownership transfer, or retention of either address is visible.

Required access depends on the selected behavior:

- A null destination requires no access through either address.
- Measurement requires readable destination storage through its first zero.
- Bounded search requires readable destination positions within the searched allowance, stopping earlier if zero is found.
- Appending requires readable source positions actually inspected and writable destination positions receiving characters or the final zero.

All address advancement must be valid. This includes advancement to the destination boundary after a fully consumed allowance, and advancement past a source zero that has been read. Neither advancement necessarily entails a read at the resulting address.

Capacity does not prove that the supplied storage exists or has suitable permissions. There is no visible overlap validation. The absence of such validation is not a general guarantee that every overlapping arrangement is valid.

## Behavior

### Selection and precedence

A null destination takes precedence over all other inputs: the result is zero, with no character-storage reads or writes.

For a non-null destination:

| Condition | Behavior |
|---|---|
| Source is null | Unbounded destination measurement, for every capacity value. |
| Source is non-null and capacity is zero or one | Unbounded destination measurement. |
| Source is non-null and capacity exceeds one | Bounded destination search, with possible appending. |

Consequently, capacity zero does not make a call access-free. A null source does not force a zero result, and a small capacity does not limit the measurement fallback.

### Measurement

Measurement returns the number of consecutive nonzero destination characters before the first zero. Destination reads proceed in increasing address order and include the terminating zero.

There are no source-character reads and no destination writes. Capacity does not constrain the traversal or result.

This behavior requires an accessible terminator and representable counting. No alternate function-level outcome is established when either requirement fails. Reading the terminating destination zero does not require advancing beyond it.

### Bounded search and the termination boundary

For bounded operation, define the allowance as:

\[
L = \text{capacity} - 1
\]

This subtraction is applied only when capacity exceeds one, so the allowance is positive and unsigned subtraction does not underflow.

At most the first \(L\) destination positions are searched. Existing nonzero characters consume the allowance.

If those positions contain no zero, the result is \(L\), subject to representable counting. In this case:

- No source character is read.
- No destination position is written.
- The destination position at offset \(L\) is not read.
- No boundary zero is inserted.

Thus, a zero exactly at offset \(L\) is not recognized. Its presence or absence does not change the outcome. The destination position advances to that boundary, so the address operation must still be valid.

If zero is found at an offset \(D<L\), appending starts at that position. The initial destination prefix remains intact, and the remaining allowance for nonzero source characters is \(L-D\).

### Append contents and result

For stable, non-overlapping storage, let:

- \(D\) be the destination prefix length found within the bounded search.
- \(S\) be the source’s initial nonzero-prefix length.
- \(L\) be the allowance.

Provided accesses and arithmetic are valid, the appended character count is:

\[
A = \min(S,\ L-D)
\]

The result is \(D+A\). The resulting destination prefix contains its original \(D\) characters followed by the first \(A\) source characters, then zero at offset \(D+A\).

This formula describes the outcome without requiring the complete source to be measured. If the allowance is reached first, no subsequent source character is read.

A final destination zero is written whenever appending has been entered, including when the first source character is zero. An empty source therefore preserves the destination’s text but still requires its existing terminator position to be writable.

The final zero is at an offset no greater than capacity minus one. Positions beyond it are not cleared. This termination guarantee applies only when a destination zero was found within the searchable allowance.

### Observable ordering

All destination-search reads precede every source-character read and every append write.

During appending, each source character is read before the destination write based on that character. For a nonzero character, counting precedes its destination write. Destination advancement follows that write, and source advancement follows destination advancement. A later source read therefore occurs after the preceding destination write.

A source zero is not transferred through the nonzero-character copy operation. Its read and the subsequent source advancement precede the separate final destination zero write. There is no required read through the advanced source address.

When the allowance is reached by a nonzero copy, advancement after that copy precedes the final zero write. The next source position is not read, even if it contains the terminator.

These distinctions matter for overlapping storage and address-validity requirements. They do not establish atomicity or synchronization guarantees.

### Overlap

There is no visible source snapshot or temporary character buffer. Destination writes may change source positions that have not yet been read, including an original source terminator.

For overlapping regions, the initial source length alone is therefore insufficient to predict the result. Later reads encounter storage after earlier writes, subject to the execution environment’s memory rules. The non-overlapping formula must not be applied as though the source were immutable.

The established behavior is the ordered access relationship, not a blanket overlap-safety promise.

### Arithmetic and failure boundaries

One signed count covers both the original destination prefix and appended nonzero characters. Numerical guarantees require every intermediate count to be representable.

The unsigned capacity does not make signed counter overflow safe. Conversely, an allowance greater than the signed maximum does not automatically invalidate a call: short prefixes may finish before that maximum is approached.

Within the representable signed range, comparison with the unsigned allowance identifies the same nonnegative count. Execution requiring a signed increment beyond that range has no portable result established here. Ordinary signed-arithmetic semantics do not supply a defined wrapping result; no target-specific wraparound or saturation behavior is established.

There is no visible allocation-extent check, explicit error reporting, or recovery mechanism. Invalid reads, writes, address advancement, or arithmetic do not have a specified function-level error result. No partial-write guarantee is established after such a precondition violation.

## Side effects

Direct writes are confined to bounded calls that find an initial destination zero within the searchable allowance. Such calls write any appended nonzero characters and then one final zero.

No destination writes occur during:

- A null-destination call.
- Measurement-only operation.
- A bounded search that consumes its allowance without finding zero.

An empty-source append still writes the final zero. An exhausted destination search does not.

The source address is used for reads and advancement, not direct writes. With overlap, destination writes may nevertheless modify storage reachable through the source address.

No external calls, explicit global-state accesses, allocation operations, or release operations are visible. Effects from concurrent mutation, device-backed storage, access faults, and asynchronous handlers are not characterized.

## Dependencies

No dependency function calls are visible, so there are no dependency function IDs to enumerate.

The function relies on caller-provided storage and the execution environment’s character, integer, address, and memory semantics. No additional validation, memory management, or recovery by another function is established.

No assumption about an unknown callee preserving globals, retaining storage, or releasing memory is needed.

## Acceptance examples

These examples are conditional static deductions, not runtime observations. Unless stated otherwise, assume stable, non-overlapping storage; valid address advancement; sufficient readable and writable storage for the described accesses; and representable counts.

Quoted text denotes a character sequence followed by zero. Offsets are measured in character storage units.

| Case | Inputs | Required result and effects |
|---|---|---|
| Null destination | Destination null; any source and capacity | Return zero without character-storage access. |
| Null source | Destination “lamp”; source null; capacity 2 | Return 4. Read through the destination zero. No writes. |
| Zero capacity | Destination “lamp”; source “Q”; capacity 0 | Return 4. Read through the destination zero; no source reads or writes. |
| Capacity one | Destination “lamp”; source “Q”; capacity 1 | Same measurement behavior: return 4 without source reads or destination writes. |
| Complete append | Destination “hi”; source “JK”; capacity 7 | Return 4. Destination becomes “hiJK”. Read the source zero; write the final destination zero at offset 4. |
| Truncated append | Destination “hi”; source “JKLM”; capacity 5 | Return 4. Destination becomes “hiJK”. Do not read source “L” or later positions. |
| Search limit reached | Destination “wxyz”; source “Q”; capacity 4 | Return 3. Read destination offsets 0–2 only. No source reads or destination writes. |
| Terminator exactly at boundary | Destination “wxy”; source “Q”; capacity 4 | Return 3 without reading the destination zero at offset 3. No source reads or writes. |
| Empty source | Destination “hi”; empty source; capacity 5 | Return 2. Read the source zero, advance the source address, and rewrite the destination zero at offset 2. |
| Minimum bounded capacity | Empty destination; source “QR”; capacity 2 | Return 1. Destination becomes “Q”. Do not read source “R”. |
| Exact source fit | Empty destination; source “Q”; capacity 2 | Return 1. Destination becomes “Q”. Do not read the source zero following “Q”. |

The two boundary-search examples deliberately have the same result and write behavior despite different contents at the unexamined boundary. Neither call establishes termination by inspection or by writing.

### Conditional overlap example

Both addresses designate the same writable region, initially containing “xy” followed by zero. Capacity is 7. Assume all required reads, writes, and address advances are valid.

The result is 6, and the region becomes “xyxyxy” followed by zero. Destination writes overwrite the original source terminator before it is read as a terminator. Subsequent source reads consume characters supplied by earlier destination writes.

A snapshot-based append would produce a different result and is not the behavior established here.

### Conditional width examples

If the signed count and unsigned capacity are both 32 bits:

- Capacity 2,147,483,648 gives an allowance of 2,147,483,647, equal to the signed maximum.
- Capacity 2,147,483,649 gives an allowance one greater than the signed maximum.

In the second case, a short destination and source can still complete with representable counting. A path requiring the count to reach that allowance has no portable outcome established by the supplied evidence.

These examples do not establish that the actual widths are 32 bits.

## Uncertainties

- **documented_only — Representation:** Exact character, integer, and pointer widths; character signedness; concrete unsigned type definition; and calling convention are unavailable.
- **documented_only — Overflow execution:** Target-specific behavior after signed-counter overflow is not established. Neither wrapping nor saturation is promised.
- **documented_only — Caller guarantees:** Allocation extents, accessible terminators, external overlap restrictions, and supported maximum lengths are unavailable.
- **documented_only — Invalid operations:** Access-fault handling, recovery, and consequences of invalid address advancement are not established.
- **documented_only — Unstable memory:** Concurrent mutation, synchronization, device-backed memory, and asynchronous effects are not characterized.
- **Inferred intent — Bounded concatenation:** The shared allowance and final zero suggest this purpose. They do not establish universally bounded reads, unconditional termination, or snapshot-based overlap semantics.

## Evidence

The supplied static representation establishes destination-null precedence, the unbounded measurement fallback, and bounded operation only for a non-null source with capacity greater than one.

Its stopping conditions distinguish an exhausted destination search from an append that reaches the same count. The former performs no writes and leaves the boundary unread; the latter writes a final zero.

The append path establishes that allowance exhaustion can prevent the next source read, including a read of the source terminator. It also establishes a final destination zero write for an empty source and source advancement after a terminating zero is read.

Interleaved source reads and destination writes, without visible snapshot storage, support the conditional overlap example. The signed count and unsigned capacity support the representability qualifications, but not concrete bit-width claims.

The absence of visible calls and ownership operations supports the dependency and ownership statements. It does not establish behavior for invalid operations or asynchronous memory changes.

No runtime tests have been performed. All acceptance examples are conditional consequences of static evidence.
