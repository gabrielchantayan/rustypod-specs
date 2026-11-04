<!-- Trusted provenance: F-7f4abd2a9add; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-7f4abd2a9add

## Purpose

F-7f4abd2a9add performs a common object operation and then selects one of two behaviors according to whether a 32-bit mode argument is zero.

- **Zero mode:** split the object operation’s 64-bit result into two 32-bit arguments, pass them through F-6ac70711bbb5, submit the resulting value to F-2926dbb9b751, and perform two ordered byte writes using separately obtained global state-area references.
- **Nonzero mode:** obtain a status-area reference associated with an opaque input, scan a zero-terminated byte sequence, and invoke F-d03768eedb4b for each selector occurrence whose corresponding status byte is nonzero.

These are operational descriptions. The evidence does not establish an application-level meaning for the modes, objects, selectors, or state bytes.

This contract describes normal execution supported by the supplied static representation. Dependency internals, memory validity, synchronization, and failure mechanisms remain unresolved where noted.

## Interface

### Inputs and output

| Item | Representation | Established use |
|---|---|---|
| Opaque input | 32-bit value | Passed unchanged to F-0bd990efcc62 in nonzero mode. Not directly used in zero mode. |
| Mode | Signed 32-bit integer | Compared with zero to select the branch and to produce the Boolean argument supplied to the indirect object operation. |
| Return | No explicit value | Completion is observable through dependency calls and memory effects, not a returned status. |

Every nonzero mode, including a negative value, selects the same branch and supplies true to the indirect operation. Zero supplies false. No other classification or range check is visible.

### Referenced layouts and widths

The function uses the following externally supplied data:

- **Service object:** returned by F-cdd33263ee89. Its initial 32-bit field supplies a dispatch-table address. The indirect callable is selected at byte offset `0x5c` within that table.
- **Submission argument:** a global value passed as the second argument to F-2926dbb9b751. Its exact declared width and meaning are not supplied.
- **State-area reference:** a global pointer-like value evaluated twice in zero mode. One evaluation supplies the destination base for the eventual offset-0 write; the other supplies the destination base for the offset-1 write.
- **Selector-sequence reference:** a global address-like value captured once in nonzero mode, after F-0bd990efcc62 returns.
- **Status-area reference:** the 32-bit result of F-0bd990efcc62. Zero disables scanning; a nonzero result is used as an address base.

The indirect object operation produces a 64-bit value. F-6ac70711bbb5 receives two 32-bit arguments and returns a 32-bit value, which becomes the first argument to F-2926dbb9b751.

Selectors and status entries are single bytes. A selector is interpreted as an unsigned 8-bit value. Selector \(k\) addresses the status byte at offset \(4+k\).

Address handling is consistent with a 32-bit target representation. The evidence does not establish equivalent behavior on a wider-address target.

### Preconditions for normal completion

Reached dependencies must return normally, and their arguments must satisfy their own contracts.

Required memory conditions include:

- A usable service object, readable dispatch information, and a callable entry at the selected dispatch offset.
- In zero mode, a writable byte at the retained offset-0 destination and a writable byte at offset `1` from the separately obtained offset-1 destination base. These destinations need not belong to the same region.
- In nonzero mode with a nonzero status-area result, a readable selector sequence extending through a reachable zero terminator.
- Readability of each status byte when its selector is processed.
- Valid index progression and address calculations for all reached accesses.

No visible check enforces these conditions.

## Behavior

### Common prefix

Both branches begin with the following ordered operations:

1. Call F-cdd33263ee89 without explicit arguments.
2. Read the returned object’s dispatch-table reference.
3. Select the indirect operation at dispatch-table byte offset `0x5c`.
4. Invoke that operation with the object and the Boolean result of testing the original mode for nonzero.

The indirect operation returns a 64-bit value. That value is consumed only in zero mode.

The service lookup and indirect invocation occur even when a later nonzero-mode status lookup returns zero. Neither mode bypasses this common prefix.

### Zero mode: conversion and submission

When the original mode is zero:

1. Extract the low 32 bits of the indirect result.
2. Extract bits 32 through 63 of that result.
3. Call F-6ac70711bbb5 with the low half first and the high half second.
4. Read the global submission argument after F-6ac70711bbb5 returns.
5. Call F-2926dbb9b751 with the 32-bit conversion result and the submission argument.

Both halves preserve their selected bit patterns. They are presented as 32-bit integer arguments; a half with its top bit set can have a negative signed interpretation. The upper half comes from an unsigned right shift of the full 64-bit result, not from sign-extending the lower half. No saturation or range validation is shown.

Because the submission argument is obtained after F-6ac70711bbb5 returns, changes that dependency makes to the global can affect the argument supplied to F-2926dbb9b751.

The return value of F-2926dbb9b751, if any, is not used.

### Zero mode: destination reads and ordered writes

After F-2926dbb9b751 returns, the supplied representation establishes this sequence:

1. Read the global state-area reference and retain that value as the **offset-0 destination base**.
2. Separately read the global state-area reference to obtain the **offset-1 destination base**.
3. Write the 8-bit value `1` at offset `1` from the offset-1 destination base.
4. Write the 8-bit value `0` at offset `0` from the retained offset-0 destination base.
5. Return.

These are two separate evaluations of the global reference, not one captured base reused for both writes. No dependency call intervenes between those evaluations or between the two writes.

If the global reference remains stable across both evaluations, the bases agree. Under that condition, and without interfering writes, the referenced region ends with byte `0` equal to zero and byte `1` equal to one.

Stability is not established unconditionally. If the evaluations yield different references, the offset-1 write targets the later reference, while the offset-0 write still targets the earlier retained reference. The final write does not obtain the global reference again.

The function does not read either destination byte before overwriting it. The writes are ordered but are not established as an atomic transaction. If both effective destinations happen to be the same byte, the later write of zero determines that byte’s final value in the absence of interference.

The opaque input is not passed to any dependency in zero mode. F-0bd990efcc62 and F-d03768eedb4b are not called.

### Nonzero mode: lookup and scan setup

After the common prefix:

1. Call F-0bd990efcc62 with the opaque input unchanged.
2. Capture the global selector-sequence reference.
3. Test whether the result of F-0bd990efcc62 is zero.

The selector-sequence reference is therefore obtained even if the lookup result is zero.

A zero lookup result causes return without reading selector bytes, reading status bytes, or calling F-d03768eedb4b. A nonzero result is retained as the status-area base.

### Nonzero mode: selector processing

Scanning starts at index zero and advances by one position at a time.

For each reached position:

- Read one unsigned selector byte from the captured sequence base plus the current index.
- If the selector is zero, stop and return.
- Otherwise, read the status byte at the retained status-area base plus \(4+k\), where \(k\) is the selector.
- If that status byte is nonzero, call F-d03768eedb4b with \(k\) first and zero second.
- Continue with the next sequence position.

Nonterminal selectors range from `1` through `255`. Corresponding status offsets range from `5` through `259`, inclusive. Selector zero terminates before any associated status access, so status offset `4` is not read by this scan.

Only the zero/nonzero distinction matters for a status byte. A status byte with bit pattern `0x80` qualifies just as `0x01` does. The selector is unsigned, so `0xff` selects `255`, not a negative index.

Each occurrence is processed independently. There is no sorting, deduplication, or early exit after the first qualifying selector.

### Visibility across callbacks

The captured sequence base and retained status-area base remain the bases for the current scan across calls to F-d03768eedb4b. Neither is visibly reacquired between entries.

Their contents are not snapshotted. Each selector and corresponding status byte is read when reached. Consequently, a callback can affect later processing by changing bytes in those retained regions, provided the regions remain valid.

Changing only the global selector-sequence reference does not redirect the current scan. Changing unread bytes within the captured sequence can change its later selectors or termination point.

A qualifying occurrence produces one call. There is no immediate reread of that occurrence’s status byte after the callback and no retry-until-cleared behavior.

## Side effects

### Direct writes

The only explicit memory writes are in zero mode, after F-2926dbb9b751 returns:

| Order | Destination | Width | Value |
|---|---|---:|---:|
| First | Separately obtained offset-1 destination base plus `1` | 8 bits | `1` |
| Second | Previously retained offset-0 destination base | 8 bits | `0` |

A shared-region final state of `[0, 1]` requires both global-reference evaluations to identify the same region and requires absence of interfering writes. It is not an unconditional postcondition.

Nonzero mode contains no explicit stores to the selector sequence or status area.

### Ownership and indirect effects

Dependencies may have effects beyond those visible in this function. No dependency is assumed to preserve globals, retain storage validity, avoid reentrancy, or leave pointed-to bytes unchanged.

No explicit allocation, release, reference-count adjustment, or ownership-transfer operation appears here. In particular:

- The service object is not explicitly released.
- The status-area reference is not explicitly released.
- Passing the conversion result to F-2926dbb9b751 does not establish whether ownership is borrowed, retained, or transferred.
- Retaining an address across a callback does not itself guarantee that its storage survives the callback.

### Errors and incomplete execution

There is no explicit error return, exception handler, rollback, null-service check, or scan-length bound.

If a dependency does not return normally, subsequent operations are not reached. The two zero-mode writes occur only after F-2926dbb9b751 returns. A failure between the writes can leave only the first write completed.

Invalid accesses have no established recovery behavior. The evidence does not identify a particular trap, exception, or diagnostic.

## Dependencies

| Dependency | Invocation condition | Arguments and result use |
|---|---|---|
| F-cdd33263ee89 | Always, first | No explicit arguments; returns the service object. |
| Indirect operation; opaque ID unavailable | Always, after service lookup | Selected through dispatch offset `0x5c`; receives the object and normalized mode; returns a 64-bit value. |
| F-6ac70711bbb5 | Zero mode | Receives low and high 32-bit halves in that order; returns the 32-bit submission value. |
| F-2926dbb9b751 | Zero mode, after conversion | Receives the conversion result and subsequently read global submission argument; result unused. |
| F-0bd990efcc62 | Nonzero mode | Receives the opaque input unchanged; zero skips scanning, nonzero supplies the status-area base. |
| F-d03768eedb4b | Each qualifying selector occurrence | Receives the unsigned selector and zero; result unused. |

The indirect operation has no supplied opaque function ID. Its dispatch location identifies selection behavior, not its implementation or full contract.

## Acceptance examples

These are conditional static acceptance cases, not executed tests. Dependencies are assumed to return normally, and all reached memory accesses and arithmetic are assumed valid.

### Zero mode with stable state reference

Suppose the indirect result is `0x89ABCDEF01234567`, and the global state-area reference remains unchanged across its two final evaluations.

Expected behavior:

- The indirect operation receives false.
- F-6ac70711bbb5 receives bit patterns `0x01234567`, then `0x89ABCDEF`.
- F-2926dbb9b751 receives the conversion result and the subsequently read submission argument.
- The state reference is evaluated once for the retained offset-0 destination and separately for the offset-1 destination.
- Byte `1` is written with one before byte `0` is written with zero.
- Without interference, that shared region ends with first two bytes `[0, 1]`.

### Submission changes the state reference

Suppose F-2926dbb9b751 changes the global reference from region A to region B, and it remains B across both subsequent evaluations.

Both writes target region B: offset `1` first, then offset `0`. No direct write to region A is required by this execution.

### State reference differs between evaluations

Suppose the first final evaluation yields region A, while the separate evaluation for the offset-1 destination yields region B.

Expected behavior:

- Write one at B plus `1`.
- Then write zero at A.
- Do not reload the reference for the final write.

This case specifies the consequence if distinct values are observed; it does not establish a synchronization mechanism permitting such a change. No `[0, 1]` postcondition for either region follows unless additional address relationships are supplied.

### Nonzero mode with zero lookup result

For mode `-1`, suppose F-0bd990efcc62 returns zero.

The indirect operation receives true, and the lookup receives the opaque input unchanged. The selector-sequence reference is captured, but no selector or status byte is read and no F-d03768eedb4b call occurs.

### Repeated selectors and termination

Suppose the sequence is `2, 7, 2, 0, 9`, status offset `6` is nonzero, and status offset `11` is zero. Assume callbacks do not change relevant memory.

F-d03768eedb4b receives `(2, 0)` twice in sequence order. Selector `7` produces no call. The terminator ends scanning before the trailing `9` is read.

### Maximum selector and high-bit status

Suppose the sequence is `255, 0`, with status offset `259` equal to `0x80`.

Exactly one callback receives selector `255` and zero. Neither byte is rejected because its high bit is set.

### Callback changes a repeated selector’s status

Suppose the sequence is `4, 4, 0`, and status offset `8` is initially nonzero. The first callback clears that byte.

The first occurrence invokes F-d03768eedb4b. The second occurrence rereads offset `8`, observes zero, and produces no callback.

## Uncertainties

The following remain **documented_only**:

- Application-level meanings of both modes and all referenced data.
- Full dependency contracts, including ownership, lifetime, failure, and reentrancy.
- Meaning of the indirect operation’s 64-bit result and its converted value.
- Exact declared representation of the global submission argument.
- Synchronization or other guarantees that could make the two state-reference evaluations agree.
- Atomicity, ordering across threads, or visibility guarantees beyond the ordering shown in the supplied representation.
- Whether retained scan storage remains valid across callbacks.
- Whether practical inputs guarantee termination before invalid address arithmetic or signed index overflow.

The scan index is represented as a signed 32-bit integer. No successful behavior is established for a sequence requiring progression beyond its representable range. Address calculations likewise have no supplied portable overflow contract.

An unterminated sequence has no established successful outcome. The absence of a visible bound does not justify assuming safe indefinite scanning or defined wraparound.

## Evidence

**Established static evidence:** the supplied representation supports the common dependency prefix, zero/nonzero branching, 64-bit result split, conversion and submission order, two separate state-reference evaluations, offset-1 write before the retained offset-0 write, selector-reference capture before the zero-result decision, retained scan bases, byte widths, status offsets, and terminator behavior.

**Inferred intent:** “service object,” “conversion,” “submission,” “state area,” “selector sequence,” and “status area” are descriptive role labels. They do not establish domain semantics.

**Unknown behavior:** dependency internals, memory ownership, external synchronization, concrete failure mechanisms, and execution outside valid arithmetic and memory preconditions are not supplied.

No runtime tests have been performed. Acceptance examples describe consequences of the static operations under their stated conditions, not observed executions.
