<!-- Trusted provenance: F-1fe3b1afff2c; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-1fe3b1afff2c

## Purpose

F-1fe3b1afff2c advances a caller-owned cursor past a narrowly defined set of ignorable bytes and percent-introduced line comments. It updates the cursor in place and returns no value.

The recognized ignorable bytes are space, carriage return, line feed, horizontal tab, form feed, and the zero byte. A percent sign begins a comment extending to a carriage return, line feed, or the supplied boundary.

The apparent intent is lexical preprocessing: position the cursor at the next byte requiring further interpretation. This intent is inferred, not an additional behavioral guarantee. In particular, the established behavior has a boundary anomaly: a comment that reaches the supplied boundary without a line ending leaves the cursor **one byte beyond that boundary**.

## Interface

### Inputs and output

The function accepts two arguments:

1. **Cursor-slot address.** An address identifying writable storage for the current cursor. The supplied representation reads and writes this slot as a 32-bit quantity and interprets its contents as a byte address.
2. **Boundary address.** A byte address used as an exclusive scanning boundary.

The function returns no value. Its output is the address written back to the cursor slot.

The boundary is not a byte count, and there is no independent length parameter. Neither argument is designated as a zero-terminated string.

### Widths and address model

The established representation uses:

- A 32-bit cursor slot.
- Byte-sized character reads.
- Address advancement in units of one byte.

No widening of the cursor slot to a host-native 64-bit pointer is justified by the supplied evidence. Any implementation or interface description must preserve the evidenced 32-bit representation or explicitly document a different external representation.

The signedness of the character type does not change the recognized byte set: every compared value is within the nonnegative ASCII range. Bytes outside that set, including high-bit bytes, are not implicitly whitespace.

### Preconditions

For ordinary, defined operation:

- The cursor slot must be readable initially and writable when the result is committed.
- The initial cursor and boundary must support meaningful ordered address comparison and the required address arithmetic.
- Every byte actually read must be accessible.
- Any required one-byte address advance must be representable within the applicable address model.

A conventional same-region arrangement, with the boundary at or after the cursor, supports the intended scanning use. However, the final increment after an unterminated comment requires special attention: if the boundary is already the one-past address of an allocated object, forming a further address can fall outside the source language’s defined pointer-arithmetic domain.

There is no explicit validation of null addresses, storage accessibility, alignment, region membership, or address overflow.

## Behavior

### Classification outside comments

At a candidate cursor strictly below the boundary, the function reads the byte there.

The following byte values are skipped:

| Byte value | Meaning |
|---|---|
| 0x20 | Space |
| 0x0D | Carriage return |
| 0x0A | Line feed |
| 0x09 | Horizontal tab |
| 0x0C | Form feed |
| 0x00 | Zero byte |

Each such byte advances the candidate cursor by one byte before scanning continues.

Any other byte, except percent, stops scanning. The final cursor points at that byte; it is not consumed.

This classification is exact. In particular:

- Vertical tab, 0x0B, is not skipped.
- No locale-dependent whitespace recognition is established.
- A zero byte does not terminate the scan.
- No decoding of multibyte text is performed.

### Comment recognition and termination

A percent byte, 0x25, encountered outside a comment starts comment scanning at that same address. There is no escape rule, quoting state, or required preceding delimiter.

Comment scanning ignores byte contents except for carriage return and line feed. Consequently:

- Spaces and tabs are ordinary comment contents.
- Zero bytes do not terminate a comment.
- Further percent bytes have no special effect.
- The first carriage return or line feed terminates the comment scan.

When a line-ending byte terminates the comment, that byte is consumed by the subsequent one-byte advance. General scanning then resumes at the following address.

For a carriage-return/line-feed pair, the carriage return ends the comment and is consumed first. The following line feed is then encountered by general scanning and skipped as an ignorable byte. Both are therefore skipped, but not as a single recognized two-byte token.

### Boundary handling

The boundary is checked before a candidate byte is read during either general scanning or comment scanning.

Normal outcomes are:

- If the initial cursor equals the boundary, no character byte is read and the same cursor is written back.
- If the initial cursor is above the boundary under the applicable ordering, no character byte is read and the same cursor is written back.
- If general scanning reaches the boundary by skipping ignorable bytes, the result is exactly the boundary.
- If a non-ignorable, non-percent byte is found below the boundary, the result is its address.

There is one distinct comment outcome:

**If comment scanning reaches the boundary without finding carriage return or line feed, the function performs a further one-byte advance and writes back the boundary plus one byte.**

The outer boundary check then prevents another character read. Thus the supplied behavior does not dereference the boundary merely because of this extra advance, but its resulting cursor is not constrained to remain within the supplied range.

This behavior must not be silently normalized to “stop at the boundary.” It is part of the supplied static behavior, subject to the pointer-arithmetic preconditions described above.

### Cursor publication

The candidate position is maintained internally during scanning. The cursor slot is not updated for each skipped byte or each comment byte.

On a normal completion path, the function writes the final candidate address to the cursor slot once. This write occurs even when the final address equals the initial address.

There is no separate success indicator, consumed-byte count, error code, or indication that a comment ended at the boundary rather than at a line ending.

## Side effects

### Observable access order

The supplied source establishes the following access ordering:

1. Read the initial 32-bit cursor representation from the caller’s cursor slot.
2. Compare the candidate address against the boundary before entering a character-scanning step.
3. Read candidate character bytes as needed for classification and comment termination.
4. Write the final cursor representation back to the original slot.

The initial byte classification uses one fetched character value for its character comparisons. Entering comment scanning causes the percent byte to be examined again as part of the comment-termination checks.

Within a comment-termination check, the carriage-return comparison precedes the line-feed comparison. The supplied expression contains a separate byte access for the latter comparison when the former does not terminate the check.

These are source-level access observations. They do not establish an exact machine-load count after compilation, nor a volatile-memory, atomicity, or memory-mapped-I/O contract.

### Memory modification and ownership

No character-buffer write is established. The only explicit memory write is the final update of the cursor slot.

The function does not allocate, free, transfer, or retain ownership of memory. It does not retain the cursor-slot address or the scanned byte address for later use.

If the cursor slot aliases the scanned region, the final cursor write can modify bytes belonging to that region. Such aliasing is not validated or prohibited by an explicit check. No broader aliasing safety guarantee is established.

No explicit global-state access or external function call appears in the supplied body.

### Errors and abnormal operation

The function has no explicit error-reporting mechanism and no recovery path.

Invalid storage, invalid pointer relationships, or unrepresentable pointer arithmetic are not converted into a documented return status. A successful cursor update is guaranteed only when execution reaches the final store under the relevant memory and address-model preconditions.

Concurrent modification of the cursor slot or scanned bytes is outside the established contract. In particular, the source does not supply synchronization or a stable-snapshot guarantee.

## Dependencies

No external function dependencies are established for F-1fe3b1afff2c.

Its behavior depends on direct byte access, 32-bit cursor storage, address comparisons, and one-byte address advancement. No allocator, character-classification routine, string-length routine, callback, or global configuration source is present.

Because there are no identified callees, this contract makes no assumptions about unknown callees preserving globals or owning memory.

## Acceptance examples

These examples are conditional static expectations, not runtime test results.

Unless otherwise specified:

- The cursor slot is separate from the scanned bytes.
- The initial cursor is at offset zero.
- The boundary is immediately after the listed bytes.
- All relevant address comparisons and address arithmetic are valid.
- The resulting offset is measured from the initial cursor.

| Input bytes | Expected final offset | Reason |
|---|---:|---|
| No bytes; cursor equals boundary | 0 | No character read; unchanged cursor is written back. |
| `A` | 0 | The first byte is neither ignorable nor percent. |
| Space, horizontal tab, `A` | 2 | Both leading ignorable bytes are skipped. |
| Zero byte, `A` | 1 | Zero is skipped rather than treated as a terminator. |
| Vertical tab, `A` | 0 | Vertical tab is not in the recognized set. |
| Space, carriage return, line feed | 3 | General scanning reaches the boundary exactly. |
| Percent, `x`, line feed, `A` | 3 | The comment and terminating line feed are consumed. |
| Percent, `x`, carriage return, line feed, `A` | 4 | Carriage return ends the comment; the following line feed is then skipped. |
| Percent, zero byte, `x`, line feed, `A` | 4 | A zero byte inside a comment does not terminate it. |
| Percent, percent, line feed, `A` | 3 | The second percent is ordinary comment content. |
| Percent, `x` | 3 | The boundary is at offset 2; the unterminated-comment path advances to offset 3. |
| Percent alone | 2 | The boundary is at offset 1; the same extra advance occurs. |

The last two examples require an address model and backing region that permit forming the stated final address. They do not authorize dereferencing that address.

Additional boundary-sensitive examples:

- **Initial cursor above the boundary:** provided the address comparison is meaningful, no character byte is read, and the initial cursor is written back unchanged.
- **Boundary excludes an immediately following line feed:** a comment reaching that boundary does not inspect the excluded line feed. It takes the unterminated-comment path and produces boundary plus one.
- **Comment ends with a line feed immediately before the boundary:** the line feed is consumed and the result equals the boundary, without the extra advance.
- **Ordinary token contains a later percent:** if scanning stops at an earlier ordinary byte, the later percent is not inspected. The function is a prefix scanner, not a whole-range comment remover.

## Uncertainties

### Established static evidence

The supplied body establishes the explicit skip set, percent-comment handling, exclusive read boundary, final cursor store, and extra advancement after a comment reaches the boundary.

### Inferred intent

Skipping lexical spacing and line comments is a plausible intended use. Neither a particular document format nor a complete tokenization policy is established.

The extra advancement on an unterminated comment may be unintended, but there is no evidence authorizing its correction in a behavioral description.

### documented_only

The following remain unresolved:

- Exact target ABI, pointer representation details, and alignment requirements beyond the evidenced 32-bit cursor slot.
- Whether callers guarantee extra backing storage beyond the scanning boundary.
- Whether callers rely on, tolerate, or reject a result beyond that boundary.
- Whether real callers allow overlap between the cursor slot and scanned storage.
- Exact compiled memory-access counts and fault behavior.
- Behavior under invalid pointers, unrelated-region comparisons, arithmetic outside the permitted address domain, or concurrent mutation.

These unknowns must not be replaced with assumed validation, saturation, error returns, or ownership rules.

## Evidence

This contract is based solely on static examination of the supplied body for F-1fe3b1afff2c.

The evidence consists of a 32-bit cursor-slot read, bounded byte scanning, six explicit ignorable-byte comparisons, a percent-triggered line-comment scan, one-byte address increments, and a final cursor-slot write. No external calls or explicit allocation operations are present.

No runtime tests have been performed. The acceptance examples are derived from the supplied control flow and remain conditional on the stated memory and address-arithmetic preconditions.
