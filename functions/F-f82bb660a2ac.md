<!-- Trusted provenance: F-f82bb660a2ac; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-f82bb660a2ac

## Purpose

F-f82bb660a2ac examines a selector-dependent bit in a small, strided table and transfers control to an unresolved indirect destination with a derived integer argument.

The established selection behavior is narrower than “find the first matching record.” The first record is examined, but its match does not stop the search and cannot distinguish a successful first-record match from no match. The second record has priority over the third. The indirect destination receives one of three values:

- **1** when the second record matches.
- **2** when the second record does not match and the third record matches.
- **0** when neither the second nor third record matches, regardless of the first record.

An initial range check invokes F-a817d8d58c10 for some selector values. The supplied evidence identifies that dependency as non-returning.

The intended application-level meaning of the table, selector, and indirect transfer is unknown. In particular, the evidence does not establish that the transfer invokes a handler stored in the selected record.

## Interface

### Inputs

The routine has two explicit inputs:

| Input | Representation | Role |
|---|---|---|
| Table address | An integer used as a memory address | Locates the first record of a strided table. |
| Selector | An unsigned integer, interpreted as signed for the initial comparison | Controls the range check and the bit tested in each record. |

The recovered representation supports a **32-bit interpretation** of the address, selector, loaded words, and selection argument. This contract uses that interpretation for concrete ranges and examples. An independent ABI declaration is not supplied; confirmation of the original machine-level widths remains documented_only.

The table is addressed using a **32-byte record stride**. Only the leading unsigned word of each examined record is visibly read:

| Record | Word location relative to table address |
|---|---:|
| First | 0 bytes |
| Second | 32 bytes |
| Third | 64 bytes |

Under the 32-bit interpretation, each word read is four bytes. No read of the remaining 28 bytes in a record is established.

### Output and termination

There is no explicit return value.

On the table-processing path, the routine invokes an unresolved indirect destination with an integer selection argument of 0, 1, or 2. If that invocation returns normally, the visible routine then returns without an explicit result.

If the initial range check calls F-a817d8d58c10, the supplied evidence says that dependency does not return. Whether its effect is termination, a fault, an exception, or some other nonlocal control transfer is unknown.

### Preconditions

For defined table-processing behavior:

- Every word that the routine actually accesses must be readable.
- Address calculations must identify the intended records without problematic overflow or address-space wraparound.
- The memory accesses must satisfy any alignment requirements of the execution environment.
- The effective shift must have established semantics on the target.
- The unresolved indirect destination must be valid and callable under the applicable calling convention.

The routine does not visibly validate these conditions.

Selectors **0 through 12** are the clearly supported selector domain: they pass the initial check and produce ordinary single-bit masks within the loaded word.

## Behavior

### 1. Apply the selector check

The initial comparison interprets the selector as signed and asks whether it is greater than 12. It does **not** perform an unsigned upper-bound check.

Under the 32-bit interpretation:

- Selectors from 0 through 12 proceed to table processing.
- Selectors from 13 through 2,147,483,647 invoke F-a817d8d58c10.
- Selectors from 2,147,483,648 through 4,294,967,295 have negative signed interpretations and therefore do not invoke that dependency at this check.

This distinction is observable and must not be replaced by a claim that every unsigned value greater than 12 is rejected.

When F-a817d8d58c10 behaves as identified and does not return, no table word is read by this routine on that path. There is no visible alternate branch for an unexpected normal return from that dependency; no reliable continuation contract is established for such a return.

### 2. Determine the bit test

Each examined record is tested using a mask formed by shifting an integer-valued one by the selector’s low eight bits.

For selectors 0 through 12, this tests the correspondingly numbered bit, with bit 0 being the least significant bit. Other bits in the loaded word do not affect the match result.

The low-eight-bit reduction is not a reduction to the word width. In particular, the visible expression does not establish a modulo-32 shift rule.

For selectors that bypass the guard through their negative signed interpretation:

- Effective shift counts from 0 through 30 have an ordinary representable single-bit interpretation under a 32-bit signed-integer model.
- A count of 31 raises a signed-shift representability issue in the recovered expression.
- Counts from 32 through 255 are outside a 32-bit operand’s shift-count range.

The supplied material does not resolve those problematic cases into portable behavior or prove a particular hardware masking rule. They remain documented_only rather than being assigned invented results.

### 3. Read the first record

The selection state initially represents 0.

The routine reads the leading word of the first record and tests the selected bit. A match assigns selection value 0; a non-match leaves selection value 0.

Consequently:

- The first-record read is real and precedes all later table reads.
- Its result does not change the eventual selection argument.
- It does not stop the search.
- It can still be observable through a fault or through the semantics of the accessed memory.

The first read must therefore not be omitted from a behavioral description merely because it does not distinguish final selection outcomes.

### 4. Read the second record

The routine next reads the leading word at table offset 32 bytes.

If the selected bit is set:

- The selection becomes 1.
- The table scan stops.
- The third record is not read.

If the bit is clear, the selection remains 0 and processing continues to the third record.

For an otherwise valid execution that reaches table processing, both the first and second words are read, even when the first record matches.

### 5. Conditionally read the third record

The leading word at table offset 64 bytes is read only if the second record did not match.

If the third record matches, the selection becomes 2. Otherwise it remains 0. No fourth record is examined.

The possible visible table-read sequences are therefore:

- First record, then second record.
- First record, then second record, then third record.

These are ordered word reads in the supplied routine, not evidence of an atomic snapshot across the table.

### 6. Invoke the unresolved destination

After scanning, the routine passes the resulting selection value to an unresolved indirect destination.

The evidence establishes the argument values and their derivation, but not:

- How the destination address is obtained.
- Whether the destination varies with the selection.
- Whether it represents a dispatch table, shared continuation, or another mechanism.
- Whether it returns.
- What state or memory it reads or modifies.

The visible routine does not pass a record pointer as its explicit selection argument.

## Side effects

### Established direct effects

The routine directly performs:

- A conditional invocation of F-a817d8d58c10.
- Two or three ordered table-word reads on the normal scanning path.
- An indirect invocation with a selection argument of 0, 1, or 2.

No direct write to table memory or other externally visible storage appears in the supplied body. No allocation or release operation is visible.

### Ownership and lifetime

The table is used as borrowed readable memory by the visible scan. There is no visible transfer of ownership and no visible requirement that the routine free it.

That observation does not constrain either dependency. The indirect destination’s behavior is unresolved, and F-a817d8d58c10 has no supplied implementation. Neither may be assumed to preserve globals, retain no pointers, or leave memory ownership unchanged.

### Memory and failure behavior

The routine does not visibly check for a null address, a short table, inaccessible memory, or misalignment. It provides no local recovery path for an invalid read.

The third word need not be accessible for a particular execution in which the second record matches, because that third word is not read. The first and second words must still be accessible.

Byte order, device-memory behavior, and concurrent mutation semantics are not established. Acceptance examples below describe the values observed by the word reads, not a prescribed byte serialization.

## Dependencies

| Dependency | Established interaction | Unknown behavior |
|---|---|---|
| F-a817d8d58c10 | Invoked before table processing when the selector’s signed interpretation exceeds 12; identified as non-returning | Termination mechanism, diagnostics, memory effects, global-state effects, and ownership effects |
| Unresolved indirect destination; opaque ID unavailable | Invoked after scanning with explicit selection argument 0, 1, or 2 | Identity, address derivation, calling convention details, state effects, ownership effects, and return behavior |

No additional dependency identity is established by the supplied evidence.

## Acceptance examples

These examples are conditional specifications, not runtime observations. Unless stated otherwise, assume 32-bit words, readable and suitably aligned memory, stable values during the scan, and a valid indirect destination.

### Second record wins

Use selector 4. Let the observed first word be 16 and the second word be 16.

The routine must read the first word and then the second word. Both match, but the first match does not stop scanning. The third word must not be read. The indirect invocation receives **1**.

### Third record wins after a first-record match

Use selector 4. Let the observed first word be 16, the second word be 0, and the third word be 16.

The routine must read all three words in order and invoke the indirect destination with **2**. The first-record match does not take priority over the third record.

### First-record-only match is indistinguishable from no match

Use selector 0. Let the observed words be 1, 0, and 0.

The routine reads all three words and invokes the indirect destination with **0**.

With observed words 0, 0, and 0, it performs the same read sequence and passes the same argument. The interface does not separately report these two situations.

### Higher supported selector

Use selector 12. Let the second observed word be 4,096.

The selected bit matches in the second record, so the indirect destination receives **1**, regardless of the first record’s matching status. Bits other than bit 12 do not affect this decision.

### Rejected ordinary out-of-range selector

Use selector 13.

The routine invokes F-a817d8d58c10 before reading the table. If that dependency satisfies its non-returning characterization, neither table scanning nor the later indirect invocation occurs.

### High-bit selector bypasses the check

Under the 32-bit interpretation, use selector 2,147,483,648. Its signed interpretation is negative, so the initial check does not invoke F-a817d8d58c10. Its low eight bits are zero, so the scan tests bit 0.

This example demonstrates the signed-check boundary. It does not establish application-level intent to support high-bit selector values.

### Third record is conditionally unnecessary

Use selector 0, with readable first and second words and an unreadable third-word address. Let the second observed word have bit 0 set.

The routine must stop scanning after the second read and pass **1** without accessing the third word. If the second word instead has bit 0 clear, a third-word access is attempted; no successful completion is promised for the unreadable address.

## Uncertainties

The following remain **documented_only**:

- Original ABI widths and calling-convention details beyond the recovered representation.
- The indirect destination’s identity and selection mechanism.
- The application meaning of selection value 0.
- Whether the first record’s outcome-neutral read is intentional.
- Shift behavior for problematic effective counts.
- Address-overflow, alignment, byte-order, and concurrent-access semantics.
- The actual effects of either dependency on globals, memory, and ownership.
- The exact non-returning behavior of F-a817d8d58c10.

An inferred intent is to validate a selector, inspect table flags, and dispatch according to a small selection value. That inference does not override the signed comparison, the mandatory first-record read, or the ambiguity of selection value 0.

## Evidence

This contract is based solely on static inspection of the supplied restricted routine representation and dependency mapping.

The established evidence covers the signed comparison against 12, the low-eight-bit shift count, the 32-byte record stride, the ordered scan of at most three leading words, the zero-valued initial selection, the stopping condition, and the final indirect invocation.

No dependency implementation, machine-level execution trace, independent ABI declaration, or runtime observation was supplied. **No runtime tests have been performed.**
