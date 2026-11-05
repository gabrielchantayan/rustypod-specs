<!-- Trusted provenance: F-4be4149abfed; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-4be4149abfed

## Purpose

F-4be4149abfed conditionally examines a byte sequence, validates its leading marker and a zero-terminated prefix, and delegates processing of the remaining bytes to another function. It returns the remaining-byte count after that delegation returns normally. Rejected inputs cause a diagnostic call followed by a return value of −1, provided the diagnostic dependency returns normally.

The established behavior is narrower than a general-purpose decoding or copying operation:

- Processing is enabled only when the fifth argument equals an address numerically derived from the fourth argument.
- The first input byte must equal 2.
- A subsequent run of nonzero bytes must contain at least eight bytes and end with a zero byte within a specified bound.
- The bytes after that zero are passed to a dependency only when their calculated count does not exceed the second argument.

The marker and minimum prefix length suggest structured-block validation. Cryptographic meaning, padding randomness, and actual copying into the first argument are not established by the supplied evidence.

## Interface

The function has five arguments and a signed integer result.

| Position | Contract role | Representation and interpretation |
|---|---|---|
| First | Opaque operation argument | A 32-bit value forwarded unchanged to the successful-path dependency. Its interpretation as an address, handle, or other value is unknown. |
| Second | Remaining-byte limit | A signed 32-bit integer compared against the calculated remaining-byte count. |
| Third | Sequence address | A byte-addressed pointer. When the entry gate matches, its first byte is read and subsequent bytes may be scanned. |
| Fourth | Extent value | A signed 32-bit integer used both to derive the entry-gate address and to calculate the scan bound. |
| Fifth | Gate address | A pointer value compared with the address obtained from the fourth argument plus one. It is not used as the address of sequence data. |

The source representation supports 8-bit bytes, 32-bit integer arguments, and a 32-bit opaque first argument. Address formation is consistent with a 32-bit address model, but the exact pointer-conversion and overflow rules are not independently established.

### Result

On the successful branch, the returned value is:

**extent − prefix length − 2**

Here, the prefix length is the number of nonzero bytes between the initial marker and the first zero byte found by the scan.

On every explicit rejection branch, the result is −1 after the diagnostic dependency returns.

The successful-path dependency’s return value is ignored. Therefore, a nonnegative return establishes that this function reached that dependency and regained control; it does not establish that the dependency completed a copy or any other intended operation successfully.

### Memory preconditions

For an ordinary defined execution:

- If the gate comparison matches, the sequence address must permit reading its first byte.
- If that byte equals 2 and the scan bound is positive, the sequence must permit every byte read before the scan terminates or exhausts its bound.
- If the successful branch is reached, arguments passed to the external dependency must satisfy that dependency’s separate, presently unknown requirements.
- Address calculations must be valid under the target execution model.

A gate mismatch does not require a sequence-byte read by this function. It does not, however, establish that the diagnostic dependency will avoid that address or other memory; the dependency’s behavior is unknown.

## Behavior

### 1. Entry gate and marker

Before reading sequence data, the function forms an address from **extent + 1** and compares that address with the fifth argument.

This is an equality test on address values. The supplied evidence does not establish that the fifth argument is an end pointer, a pointer into the sequence, or a pointer that will be dereferenced. Its relationship to the third argument is not checked.

If the addresses differ:

- No sequence byte is read directly by this function.
- The function selects diagnostic detail **0x6b**.
- It invokes the diagnostic dependency and then returns −1 if that call returns.

If the addresses match:

1. The first byte at the sequence address is read.
2. The scan position becomes the address immediately after that byte.
3. The byte value is compared with 2.

A first byte other than 2 selects the same diagnostic detail, **0x6b**. No subsequent sequence byte is scanned on that path.

### 2. Scan bound and prefix

After a matching gate and marker, the scan bound is calculated as:

**bound = extent − 1**

The prefix count starts at zero. Scanning begins immediately after the marker.

For a positive bound, bytes are examined in increasing address order:

- Each nonzero byte increments the prefix count.
- A zero byte stops the scan without being included in that count.
- Once the count reaches the bound, no further sequence byte is read by the scan.

Consequently, when the bound is positive, the scan reads at most `bound` bytes after the marker.

A zero in the last permitted position is distinguished from an exhausted scan. For example, if the bound is nine, eight nonzero bytes followed by zero yield a prefix count of eight and a found terminator. Nine nonzero bytes exhaust the bound; the function does not read a tenth byte to look for a terminator.

If a zero is found, the pointer retained for the later dependency identifies the byte immediately after that zero.

### 3. Rejection precedence

After scanning, rejection checks have this order:

1. **Prefix count equals the bound:** diagnostic detail **0x71**.
2. **Prefix count is less than eight:** diagnostic detail **0x67**.
3. Otherwise, calculate the remaining-byte count and compare it with the second argument.

The ordering is observable. An exhausted scan with fewer than eight nonzero bytes receives **0x71**, not **0x67**.

A zero bound causes no scan-byte reads. The initial count of zero equals that bound, so the detail is **0x71**.

A negative bound also causes no scan-byte reads. The initial count is not equal to a negative bound, but is less than eight, so the detail is **0x67**. These statements apply only where the preceding arithmetic and address formation have defined target behavior.

### 4. Remaining-byte calculation

When a zero has been found after at least eight nonzero bytes, the function calculates:

**remaining count = bound − (prefix count + 1)**

The extra one accounts for the zero terminator. The initial marker was already excluded when the bound was formed.

For ordinary non-overflowing arithmetic on this branch:

- The prefix count is at least eight.
- The prefix count is strictly less than the bound.
- The remaining count is nonnegative.
- A zero remaining count is permitted.

The function compares the remaining count with the second argument using signed integer ordering.

If the remaining count exceeds that argument, it selects diagnostic detail **0x6d**, calls the diagnostic dependency, and returns −1 if the call returns.

There is no separate rejection of a negative second argument. Because the remaining count on this branch is nonnegative, a negative second argument fails this comparison.

### 5. Successful delegation

If the remaining count is no greater than the second argument, the function invokes the successful-path dependency with these five values, in order:

1. The unchanged first argument.
2. The address immediately after the zero terminator.
3. The remaining count.
4. The calculated bound.
5. The original extent value.

After that call returns normally, the function returns the remaining count.

The function does not inspect payload bytes after the zero terminator before making this call. It also does not inspect the dependency’s return value or independently confirm any destination contents.

## Side effects

### Direct reads

The function’s direct sequence reads occur in this order:

1. No sequence read before the gate comparison.
2. One marker-byte read if the gate matches.
3. Increasing-address prefix reads only if the marker equals 2 and the bound permits scanning.

It does not directly read the payload after the terminator.

### Direct writes and state

No direct write to caller-supplied sequence memory or to memory designated by the first argument is shown. No explicit global read or write is shown.

These observations do not constrain the dependencies. Either dependency may access or modify memory, globals, external state, or diagnostic state. Those effects are not described by the supplied body.

### Ownership

No allocation, release, or explicit ownership transfer is shown within this function.

The function passes a pointer into the sequence to the successful-path dependency. Whether that dependency retains the pointer, modifies the sequence, transfers data elsewhere, or releases any allocation is unknown. No ownership guarantee should be inferred solely from this caller.

### Errors and abnormal termination

Explicit validation failures use a shared diagnostic call. They do not directly return different error numbers; their common return is −1.

Memory faults, arithmetic behavior outside the established range, dependency failures, nonlocal control transfers, and process termination are not converted into documented recoverable errors by this function.

## Dependencies

### F-4fc89ec41a6d

Established role: diagnostic-path dependency.

Every explicit rejection calls it with five arguments:

| Position | Value |
|---|---|
| First | 4 |
| Second | 0x71 |
| Third | Selected diagnostic detail |
| Fourth | 0 |
| Fifth | 0 |

The selected detail is:

| Condition | Detail |
|---|---|
| Gate mismatch or marker not equal to 2 | 0x6b |
| Prefix count equals the scan bound | 0x71 |
| Non-exhausted prefix shorter than eight bytes | 0x67 |
| Remaining count exceeds the signed limit | 0x6d |

The meaning of these values within the dependency is unknown. Logging, error-stack mutation, global preservation, allocation, and release behavior are not established.

### F-000000000001

This is a document-local opaque identifier for the successful-path external dependency; its published identity was not supplied.

Its established interface at this call site consists of the five arguments listed under “Successful delegation.” The caller ignores any returned value.

A data-transfer role is inferred from the argument arrangement, but copying, overlap handling, destination validity, ownership, and failure reporting remain undocumented.

## Acceptance examples

These are conditional static acceptance criteria, not executed tests. Unless stated otherwise, assume valid readable memory, ordinary non-overflowing arithmetic, a matching gate address, and dependencies that return normally.

| Scenario | Expected behavior |
|---|---|
| Gate address differs from the address formed by extent plus one | No direct sequence read; diagnostic detail 0x6b; return −1. |
| Gate matches, but the first byte is 1 | Read the marker only; diagnostic detail 0x6b; return −1. |
| Extent is 1 and the marker is 2 | Bound is zero; no prefix read; diagnostic detail 0x71; return −1. |
| Extent is 0 and the marker is 2 | Bound is −1; no prefix read; diagnostic detail 0x67; return −1. |
| Extent is 5, marker is 2, and all four permitted following bytes are nonzero | Exhaust the bound; diagnostic detail 0x71, despite the count being below eight; return −1. |
| Extent is 10, marker is 2, and the next byte is zero | Prefix count is zero; diagnostic detail 0x67; return −1. |
| Extent is 10, marker is 2, followed by eight nonzero bytes and a zero | Remaining count is zero. With a nonnegative limit, call F-000000000001 using the address immediately after the zero, count zero, bound nine, and extent ten; return zero. |
| The preceding zero-payload case uses a limit of −1 | Diagnostic detail 0x6d; no successful-path call; return −1. |
| Extent is 13, marker is 2, followed by eight nonzero bytes, zero, and three payload bytes; limit is three | Call F-000000000001 with count three and the address of the first payload byte; return three. |
| The preceding three-byte case uses a limit of two | Diagnostic detail 0x6d; return −1. |
| Extent is 10, marker is 2, followed by nine nonzero bytes and then zero | Read only the nine permitted prefix bytes; do not read the later zero; diagnostic detail 0x71; return −1. |

For the three-byte successful case, changes to the three payload byte values do not alter this function’s direct validation decisions. Their effect inside F-000000000001 is unknown.

## Uncertainties

### Established static evidence

The gate comparison, marker value, bounded scan, rejection ordering, diagnostic arguments, successful-call arguments, and explicit return values are established by the supplied function body.

### Inferred intent

The sequence appears to comprise a one-byte marker, a nonzero prefix of at least eight bytes, a zero delimiter, and a remaining payload. The second argument appears to serve as a capacity limit.

These interpretations do not establish a particular encoding standard or a destination-copy guarantee.

### documented_only: arithmetic boundaries

The extent participates in both addition by one and subtraction by one. At signed 32-bit extremes, those operations raise overflow questions.

The supplied representation does not establish whether execution wraps, traps, or is governed by another language or machine-level rule. In particular, the address gate at the maximum signed extent and bound formation at the minimum signed extent must not be assigned ordinary mathematical behavior without additional evidence.

The successful remaining-count calculation does not overflow under the ordinary validated scan constraints.

### documented_only: dependency behavior

Neither dependency’s implementation is supplied. Global preservation, memory ownership, aliasing restrictions, thread safety, retention of pointers, and error semantics remain unknown.

### documented_only: pointer model

The comparison uses an integer-derived address, not an established relationship between two pointers into the same allocation. Exact integer-to-pointer conversion, valid address ranges, and behavior of extreme pointer arithmetic require target-specific evidence.

## Evidence

This contract is based solely on static examination of the supplied function body and the supplied dependency mapping.

No runtime tests, dependency inspection, memory-access tracing, or binary-level verification have been performed. Acceptance examples express consequences of the visible branches under their stated preconditions; they are not observed execution results.
