<!-- Trusted provenance: F-841ca88dca4b; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-841ca88dca4b

## Purpose

F-841ca88dca4b conditionally consumes values through F-9c7d16339f0f, updates a caller-supplied output region, and advances a 32-bit field in a caller-supplied state region.

The function first checks a required marker. On a matching marker, it acquires an initial selector, consumes a variable-length sequence, records a width-dependent value, and acquires a presence-like value. A zero presence-like value ends processing successfully. A nonzero value enables additional conditional reads and output updates.

**Established behavior:** three return values are visible: 0, 1, and 2. Output updates are incremental, not transactional. Return 2 can follow substantial input consumption and output modification.

**Inferred intent:** the dependency likely extracts a requested number of bits from a stream, and the function likely parses a structured header. Neither interpretation is necessary to describe the established behavior. In particular, the supplied material does not establish that every dependency result fits the requested width.

## Interface

The function has three arguments:

1. **State region:** an address represented by a 32-bit argument. It is passed to every dependency invocation. A 32-bit field at offset +4 is explicitly incremented twice on paths that pass the initial marker check.
2. **Configuration region:** an address represented by a 32-bit argument. Individual bytes at the offsets listed below control dependency requests and branch selection.
3. **Output region:** a writable byte-addressed region. Writes can reach offset +27.

The return value is 32 bits:

| Value | Established meaning |
|---|---|
| 0 | Either the stored presence byte was zero, or the extended processing path reached its final state-field copy. |
| 1 | The first dependency result did not equal the required marker. |
| 2 | The presence byte was nonzero, but a configuration check selected or reached the mode associated with byte value 2 before extended processing completed. |

These are behavioral distinctions, not established public error names.

### Configuration layout

All listed configuration accesses are one byte wide.

| Offset | Role in this function |
|---|---|
| +44 | Mode byte, compared with 0 and 2 at several distinct points. |
| +48 | Width argument for the first variable-width dependency request. |
| +86 | Additional eligibility byte; compared with 2 when the output selector is 3. |
| +131 | Width argument for the extended variable-width dependency request. |
| +142 | Enable-like byte; compared with zero. |

Repeated accesses are not an established snapshot of configuration at entry.

### Output layout

| Offset | Width | Assigned value or role |
|---|---:|---|
| +0 | 1 byte | Low 8 bits of the result requested with width 2. |
| +4 | 2 bytes | Low 16 bits of the first configuration-width result. |
| +6 | 1 byte | Low 8 bits of a result requested with width 1; determines the early-success branch. |
| +7 | 1 byte | Conditional width-1 result, or explicit zero. |
| +9 | 1 byte | Separately conditional width-1 result, or explicit zero. |
| +10 | 1 byte | Low 8 bits of an extended result requested with width 3. |
| +12 | 2 bytes | Low 16 bits of the extended configuration-width result. |
| +14 | 1 byte | Constant 1, or low 8 bits of another width-3 result. |
| +15 | 1 byte | Constant 1, or a derived shifted value narrowed to 8 bits. |
| +16–17 | 2 bytes | Either explicit bytes 32 and 0, or a derived value stored as a 16-bit unit. |
| +18 | 1 byte | Additional width-3 result, acquired only when a later selector read equals 2. |
| +24 | 4 bytes | A late copy of the state field at +4. |

No direct write is shown to output offsets +1–3, +8, +11, +19–23, or beyond +27. Fields omitted by a branch are not initialized by that branch.

### Preconditions and representation limits

For ordinary completion, each accessed address must be valid for its access width, and the dependency must return normally. The supplied material contains no pointer, capacity, alignment, or configuration-width validation.

A 28-byte output region covers the maximum directly accessed extent. A contiguous configuration region extending through offset +142 covers all visible configuration accesses. Actual paths can access less.

Disjoint storage is not checked. If the regions overlap, direct stores can affect later configuration reads or state updates. Dependency side effects can also affect later observations. Exact alias-sensitive behavior additionally depends on the platform’s multi-byte representation.

## Behavior

### 1. Marker check

The first operation is a call to F-9c7d16339f0f using the state argument and a width argument of 32.

Its full visible result is compared with decimal 438. A mismatch returns 1 immediately.

On this path, F-841ca88dca4b performs no direct output store or direct state-field increment. The dependency may nevertheless have changed memory or consumed input; its effects are unknown.

### 2. Initial selector and variable-length consumption

After a marker match:

1. A width-2 dependency result is narrowed to 8 bits and stored at output +0.
2. The function repeatedly requests width 1.
3. Repetition continues only while the full dependency result equals 1.

The terminating result is consumed but is not stored in the output. Any full result other than 1 terminates the loop; the function does not require that result to be zero.

There is no visible iteration limit. If the dependency keeps returning 1, the function does not proceed to its explicit state increments.

### 3. First state increment, variable-width field, and second increment

After the loop terminates:

1. The function reads the 32-bit state field at +4, adds one, and writes it back.
2. It reads the configuration byte at +48 and uses that value as the next dependency width.
3. It stores the low 16 bits of the result at output +4.
4. It reads the state field at +4 again, adds one, and writes it back again.

The second increment operates on the then-current field value. It is not established to use the value produced by the first increment: the intervening dependency call may modify the field.

The field is represented as a signed 32-bit quantity in the supplied material. Behavior at signed overflow is not established as a portable arithmetic rule; see **Uncertainties**.

### 4. Presence byte and short success

The function requests width 1 and stores the result’s low 8 bits at output +6.

If the stored byte is zero, it returns 0 immediately. It does not directly initialize the later output fields or copy the state field to output +24.

This decision uses the narrowed byte, unlike the earlier repetition loop. A dependency result with low byte zero takes this return path even if the full result is nonzero.

### 5. First conditional flag

When output +6 is nonzero, the function chooses the value for output +7.

It writes zero without a dependency call when the current configuration mode byte equals 2.

Otherwise, it reads output +0 and permits a width-1 dependency call when either:

- the selector equals 1; or
- the selector equals 3 and configuration +86 equals 2.

When permitted, it stores the low 8 bits of that dependency result at output +7. Otherwise, it writes zero there.

Configuration +86 is only needed for the selector-3 condition. The selector is reread from output; it is not an immutable saved copy of the original width-2 result.

### 6. Second conditional flag

The function then determines output +9 using short-circuit evaluation:

1. It reads configuration +142. If zero, it writes zero.
2. Otherwise, it reads configuration +44. If nonzero, it writes zero.
3. Otherwise, it reads output +0. If greater than 1 as an unsigned byte, it writes zero.
4. Otherwise, it requests width 1 and stores the low 8 bits at output +9.

Thus the dependency call occurs only when the enable-like byte is nonzero, the mode byte is zero, and the selector byte is 0 or 1 at those reads.

### 7. Mode gating and extended acquisition

The function reads configuration +44 again.

- If it equals 2, the function returns 2.
- Otherwise, it requests width 3 and stores the low 8 bits at output +10.

After that dependency call and store, it reads configuration +44 again.

- If it now equals 2, the function returns 2.
- Otherwise, extended processing continues.

The second mode check is behaviorally distinct. A dependency side effect or overlapping output store can make its result differ from the preceding check.

On continuation, the function reads configuration +131, uses it as a dependency width, and stores the low 16 bits of the result at output +12.

### 8. Selector-dependent derived fields

After the extended variable-width call, the function rereads output +0.

If it equals zero, four one-byte stores occur in this order:

1. Output +14 receives 1.
2. Output +15 receives 1.
3. Output +16 receives 32.
4. Output +17 receives 0.

There is no additional selector-width dependency call on this branch.

If the selector is nonzero:

1. The function requests width 3.
2. It narrows the result to an unsigned 8-bit value and stores it at output +14.
3. It forms a shift count by subtracting one from that byte and retaining the low 8 bits of the subtraction.
4. It shifts an integer value of 1 left by that count and stores the low 8 bits at output +15.
5. It shifts integer value 16 left by the unsigned byte value and stores a 16-bit result at output +16.
6. It rereads output +0. If that byte equals 2, it requests width 3 and stores the low 8 bits at output +18.

The shift expressions do not establish defined results for every possible dependency return. In particular, a narrowed result of zero produces a first shift count of 255, not zero.

The final selector comparison occurs after another dependency invocation and several stores. It can therefore observe a selector different from the one that selected the nonzero branch.

### 9. Final state copy

After either selector-dependent branch completes, the function reads the current 32-bit state field at +4 and stores that value at output +24. It then returns 0.

This is a late snapshot, after all dependency calls on the completed path. It is not necessarily the entry value plus two.

## Side effects

The function’s direct side effects are the output stores described above and two explicit read-modify-write operations on state +4.

There is no direct allocation, release, ownership transfer, logging, or cleanup operation in the supplied body. That does not establish the absence of such behavior inside F-9c7d16339f0f.

Output mutation is incremental:

- Return 1 follows only the marker dependency call.
- The short return 0 follows writes to output +0, +4, and +6 and both explicit state increments.
- Return 2 follows additional flag writes and may also follow the write to output +10.
- The extended return 0 additionally writes the later fields and the final state snapshot.

There is no rollback or comprehensive output clearing. Existing bytes survive wherever the selected path performs no store, subject to dependency effects and aliasing.

The function contains no visible handling for dependency exceptions, faults, nonlocal exits, or failure indications. A dependency result outside an expected bit range is generally consumed through the comparisons, narrowing, and arithmetic described above.

## Dependencies

### F-9c7d16339f0f

Every invocation receives the original state argument and a width argument.

In path order, the requests are:

1. Width 32.
2. Width 2.
3. Width 1, repeated through the first full result not equal to 1.
4. Width taken from configuration +48.
5. Width 1 for the presence byte.
6. Optional width 1 for output +7.
7. Optional width 1 for output +9.
8. Width 3 for output +10, unless a preceding mode check returns 2.
9. Width taken from configuration +131, unless the second mode check returns 2.
10. Width 3 on the nonzero-selector derived-field branch.
11. Optional width 3 when the final selector read equals 2.

**Unknown:** valid width range, result-range guarantees, input consumption rules, failure signaling, mutations to state or other memory, and allocation or ownership behavior.

No contract here assumes that this dependency preserves configuration, output, globals, or the state field.

## Acceptance examples

These examples are conditional static expectations, not runtime test results. Unless stated otherwise, assume valid disjoint storage, normal dependency returns, no relevant dependency memory mutations, and representable state increments.

### Marker mismatch

If the first result is 437, the function returns 1 after one dependency invocation. It performs no direct output store and no explicit state increment.

### Short successful path

Suppose the results are:

- marker: 438;
- initial selector: 2;
- repetition sequence: 1, 1, 0;
- first variable-width result: 4660;
- presence result: 0.

The function returns 0. Output +0 is 2, the 16-bit value at +4 is 4660, and output +6 is zero. The state field has received two increments. Output +24 is not directly written.

### Full-result versus narrowed-byte decisions

A repetition result of 257 ends the loop because it is not equal to 1.

A later presence result of 256 stores zero at output +6 and causes the short return 0. These two decisions intentionally have different comparison widths in the visible behavior.

### Stable mode 2

With nonzero presence and a stable mode byte of 2, output +7 and +9 are both set to zero. The function returns 2 without requesting the width-3 value for output +10.

### Mode changes during extended acquisition

If the mode is not 2 before the output +10 call, but that dependency call changes it to 2, output +10 is still stored. The following mode check returns 2. Output +12 and the final state snapshot are not directly written.

### Zero-selector extended completion

With nonzero presence, stable mode 0, selector 0, and successful extended acquisition:

- output +7 is zero;
- output +9 is acquired only if configuration +142 is nonzero;
- output +14 through +17 become the bytes 1, 1, 32, and 0;
- output +18 is not directly written;
- output +24 receives the final state-field value;
- the return value is 0.

### Defined derived-value case

On the nonzero-selector branch, if the derived width-3 result is 3, output +14 becomes 3, output +15 becomes 4, and the 16-bit value at output +16 becomes 128.

If the final selector read equals 2, an additional width-3 call supplies output +18.

## Uncertainties

- **documented_only — Dependency semantics:** stream extraction is inferred intent. No range, preservation, ownership, or error contract is supplied.
- **documented_only — Shift boundary behavior:** a derived byte of zero yields a shift count of 255 for output +15. Other out-of-range dependency results can also produce excessive counts or signed-shift problems. Exact machine behavior is not established.
- **documented_only — Counter overflow:** the visible 32-bit addition does not establish a portable signed-overflow policy. Acceptance examples avoid that boundary.
- **documented_only — Multi-byte representation:** byte order and alignment requirements are not explicitly provided. The four explicit byte stores on the zero-selector branch have established byte order; native-width stores do not independently establish a byte sequence.
- **documented_only — External mutation:** the dependency may alter memory subsequently reread by this function. Stable-configuration examples are conditional, not a preservation guarantee.
- **documented_only — Domain meanings:** the semantic names of the marker, selector, modes, flags, derived quantities, and state field are unavailable.
- **documented_only — Invalid storage:** behavior for null, inaccessible, undersized, or improperly aligned regions is not specified by visible checks.

## Evidence

This contract is based on static inspection of the supplied body for F-841ca88dca4b and the supplied mapping of its sole visible dependency to F-9c7d16339f0f.

Established evidence includes call ordering, branch conditions, access widths and offsets, narrowing conversions, explicit state updates, output-store ordering, arithmetic expressions, and return values.

Header parsing, bit extraction, and field meanings are inferred intent only. Dependency implementation, surrounding callers, platform representation guarantees, and runtime traces were not supplied. No runtime tests have been performed.
