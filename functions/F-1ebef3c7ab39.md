<!-- Trusted provenance: F-1ebef3c7ab39; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-1ebef3c7ab39

## Purpose

F-1ebef3c7ab39 conditionally clears a bit in a caller-supplied object and searches an associated forward-linked structure for an entry whose one-byte tag equals a supplied selector. It then either transfers control to F-feee9156a257 or invokes two other dependencies and stores a four-byte result in the object.

This purpose is established by the supplied static representation. Describing the operation as removal, cancellation, or notification would add intent not established by that representation. In particular, the available evidence does not establish that F-feee9156a257 frees an entry, reports an error, or performs any other specific operation.

The function has no explicit return value. Its observable outcomes are normal return without action, normal return after dependency calls and a result write, or a call to a dependency annotated as non-returning.

## Interface

### Inputs

The function accepts four arguments:

| Position | Representation | Role established by the evidence |
|---|---|---|
| First | 32-bit address-bearing value | Identifies the object containing the head reference, bit field, and result field. |
| Second | Unsigned 32-bit integer | Supplies both the shift selector and the value compared with entry tags. |
| Third | Four-byte value | Not referenced by the shown body. |
| Fourth | Four-byte value | Copied to local storage, whose address is passed to two dependencies on the nonempty, normally completing path. |

The first argument is represented as an integer but used as an address. The supplied representation supports a 32-bit address model; it does not independently establish a portable interface for other address widths.

The third argument has no demonstrated effect within this function. That does not establish a broader calling-convention guarantee concerning its ABI-level presence.

### Object layout

Offsets are byte offsets relative to the first argument:

| Offset | Width | Use |
|---|---:|---|
| +4 | 4 bytes | Head reference. Zero causes immediate return. |
| +8 | 4 bytes | Bit field, read and conditionally updated. |
| +12 | 4 bytes | Result field, written after F-11922d899a44 returns. |

No access to offset +0 is shown. No size, type, initialization rule, or ownership rule for other object fields is established.

### Entry layout

The head reference and subsequent nonzero links are treated as entry addresses:

| Offset | Width | Use |
|---|---:|---|
| +4 | 4 bytes | Next-entry reference; zero terminates traversal. |
| +19 | 1 byte | Tag compared with the full unsigned selector. |

No other entry fields are read or written directly.

### Preconditions

For defined execution along a selected path:

- The object’s head-reference field must be readable.
- If that reference is nonzero, the bit field must be readable.
- If the selected bit is set, the bit field must also be writable, and every entry field actually visited must be readable.
- A matching non-head entry requires its predecessor’s next-reference field to be writable.
- Normal completion after the dependency calls requires the object’s result field to be writable at that time.
- The dependency calls must accept the arguments actually supplied.

The function does not validate these conditions. In particular, a zero first argument is not treated as an empty object: the function first attempts to read the field at offset +4.

## Behavior

### 1. Read the head reference

The first demonstrated memory access is a four-byte read at object offset +4.

If this value is zero, the function returns immediately. On this path it does not read the bit field, traverse entries, call any dependency, or write the result field.

A nonzero head reference is retained for the subsequent entry operations.

### 2. Form a mask and prepare local argument storage

For a nonempty object, the function forms a mask by right-shifting the unsigned 32-bit value with only its highest bit set. The shift count is the selector’s low eight bits.

For low-eight-bit counts from 0 through 31, the mask has exactly one set bit:

- Count 0 selects bit 31.
- Count 1 selects bit 30.
- Count 31 selects bit 0.

The representation does not establish a defined portable result for counts from 32 through 255. It is therefore not justified to assume those counts wrap modulo 32, produce zero, or behave in any other particular way.

The fourth argument is copied into a four-byte local cell. This cell is distinct from the object’s result field. Its address is later supplied to dependencies if execution reaches the completion sequence.

### 3. Test and, when selected, clear the bit

The function reads the object’s bit field and tests its intersection with the mask.

If the intersection is zero, it skips all entry inspection and proceeds directly to the completion sequence.

If the intersection is nonzero, it reads the bit field again and writes back that value with the selected mask bit cleared. The update is an ordinary read-and-write sequence; no atomic operation or synchronization is shown.

This write happens before:

- Reading the head entry’s next-reference field.
- Reading the head entry’s tag.
- Calling F-feee9156a257.
- Calling either completion dependency.

The second bit-field read matters to the access contract: the shown body does not reuse only the value previously read for the branch test. No safe concurrent-update behavior is established.

### 4. Inspect the head entry

After clearing the bit, the function reads the head entry’s next-reference field. It then reads the head entry’s one-byte tag and compares it with the entire unsigned 32-bit selector.

The tag is not compared merely with the selector’s low byte. For selectors greater than 255, a one-byte tag cannot equal the selector.

If the head tag matches, the function calls F-feee9156a257. The supplied representation shows no explicit argument for this invocation and annotates it as non-returning.

Before that call:

- The selected object bit has already been cleared.
- The head’s next reference has already been read.
- The object’s head reference has not been rewritten by this function.
- Neither completion dependency has been called.
- The result field has not been written by this function.

The static representation does not establish behavior if this invocation unexpectedly returns.

### 5. Search subsequent entries

If the head tag does not match, traversal begins with the next reference already read from the head.

For each nonzero current entry:

1. The function reads its one-byte tag.
2. It compares that tag with the full unsigned selector.
3. If the tag does not match, that entry becomes the predecessor, and its next reference is read for the next iteration.
4. If the tag matches, the function reads the matching entry’s next reference and writes it into the predecessor’s next-reference field.

Thus, on a non-head match, the matching entry is bypassed in the predecessor’s link before F-feee9156a257 is invoked.

The function then calls F-feee9156a257 with the matching entry address shown as an explicit argument. This invocation is also annotated as non-returning.

The representation includes a zero check on the matching entry address after the predecessor-link write. Under the shown control flow, that address has already been required to be nonzero to enter the iteration. The check therefore supplies no reachable normal-return alternative under ordinary execution of the shown body.

Only the first matching entry reached in traversal order is handled. Because the matching path transfers to a non-returning dependency, later entries are not searched on that path.

If traversal reaches a zero next reference without a match, the function proceeds to completion. The earlier bit clear is not undone.

A cycle consisting only of nonmatching entries has no shown termination mechanism.

### 6. Complete the normally returning nonempty path

This sequence is reached when:

- The head reference was nonzero and the selected bit was clear; or
- The selected bit was set, but no visited entry matched.

The sequence is:

1. Call F-c082dad756a2 with the address of the four-byte local cell initialized from the fourth argument.
2. After that call returns, call F-11922d899a44 with the address of the same local cell.
3. After the second call returns, store its four-byte result at object offset +12.
4. Return without an explicit value.

The first dependency’s return value, if any, is not used by the shown body.

The local cell may have been changed by F-c082dad756a2 before F-11922d899a44 receives its address. The second dependency is not guaranteed to receive the original fourth-argument value.

The final store targets the result field derived from the original first argument. No intervening reread of the object’s head reference or bit field is shown.

## Side effects

### Direct writes

The function directly performs only these demonstrated writes:

- Initialization of the four-byte local cell from the fourth argument, on the nonempty path.
- Clearing the selected bit in the object’s bit field, when the initial test finds it set.
- Replacing a predecessor’s next reference with the matching non-head entry’s next reference, when such a match is found.
- Storing F-11922d899a44’s four-byte return value in the object’s result field, after both completion calls return.

It does not directly clear the removed entry’s next reference, overwrite its tag, or change the object’s head reference.

### Ownership and lifetime

No allocation or explicit deallocation is shown.

Bypassing an entry does not, by itself, establish that its storage is released or that ownership transfers. The role of F-feee9156a257 remains unknown.

Both completion dependencies receive an address into this function’s local storage. That storage is valid during the calls shown; no permission to retain the address beyond this invocation is established. Whether either dependency retains it is unknown.

No assumption may be made that any dependency preserves globals, object contents, entry storage, or allocation state. If a dependency invalidates storage required by a later operation, the supplied body provides no recovery mechanism.

### Errors and partial effects

No explicit error code, validation failure, rollback, or exception handling is shown.

The bit clear and predecessor-link update occur before the relevant dependency call. If that call terminates execution or otherwise prevents normal completion, those earlier writes have already occurred.

Likewise, if a completion dependency does not return, the final result-field store is not reached.

## Dependencies

| Opaque function ID | Invocation established here | Unknown behavior |
|---|---|---|
| F-c082dad756a2 | Called first in the completion sequence with the address of the local four-byte cell. Its return value is not used. | Accepted data format, local-cell mutations, other side effects, ownership behavior, and failure behavior. |
| F-11922d899a44 | Called second with the same local-cell address. Its four-byte return value is stored at object offset +12. | Meaning of the result, accepted cell state, side effects, and failure behavior. |
| F-feee9156a257 | Called on a matching head with no explicit argument shown, or on a matching later entry with that entry address shown. Both calls are annotated as non-returning. | Actual signature, reason for the differing argument presentations, effects on storage, and the mechanism preventing return. |

The two displayed forms of F-feee9156a257 must not be normalized into a single invented calling contract.

## Acceptance examples

These examples are conditional static expectations, not runtime test results. Unless otherwise stated, referenced storage is valid and completion dependencies return normally.

### Empty object

Given a zero head reference:

- The function returns after reading that reference.
- The bit field and result field remain untouched by this function.
- No dependency is called.

This remains the shown path regardless of the selector or fourth argument.

### Selected bit absent

Given a nonzero head reference, selector 0, and bit 31 clear:

- The bit field is read for the test but not written.
- No entry tag or entry next reference is read by this function.
- Both completion dependencies are called in order.
- The result field receives the four-byte return value from F-11922d899a44.

The nonzero head need not be dereferenced on this path by this function itself.

### Head match

Given selector 0, bit 31 set, and a head tag of 0:

- Bit 31 is cleared.
- The head’s next reference is read before its tag.
- F-feee9156a257 is called with no explicit argument shown.
- No direct head-reference rewrite or completion call precedes that invocation.

Normal completion afterward is not established.

### Match after the head

Given selector 31, bit 0 set, a nonmatching head, and a later entry tagged 31:

- Bit 0 is cleared.
- Entries are visited in link order.
- The first matching entry’s predecessor is updated to reference the matching entry’s successor.
- F-feee9156a257 is then called with the matching entry address.
- This function has not written the result field before that call.

### Set bit, no match

Given a valid shift count, the corresponding bit set, and a finite list with no matching tag:

- The selected bit is cleared.
- The list is traversed to its zero terminator.
- No entry link is rewritten.
- Both completion dependencies run, followed by the result-field store.

### Full-width comparison differs from shift selection

Given selector 256:

- Its low eight bits are zero, so the demonstrated mask calculation selects bit 31.
- No one-byte tag can equal the full selector value 256.
- If bit 31 is set and traversal is finite, the bit is cleared and traversal reaches completion without a tag match.

This example does not require resolving an oversized shift, because the extracted shift count is zero.

### Completion dependency changes the local cell

If F-c082dad756a2 modifies the local cell and returns, F-11922d899a44 receives the address of that modified cell. The body does not restore the original fourth argument between calls.

## Uncertainties

The following are **documented_only**:

- Shift behavior when the selector’s low eight bits are 32 through 255.
- The real signature and semantics of F-feee9156a257.
- Whether its non-returning annotations accurately describe all executions.
- Allocation, release, and ownership rules for the object and entries.
- Side effects of all dependencies, including global changes and pointer retention.
- The semantic meaning of the bit field, entry tags, fourth argument, and stored result.
- Required alignment and byte order.
- Intended synchronization, if objects can be shared concurrently.
- Intended behavior for malformed links, cycles, invalid addresses, or dependency-induced invalidation.

A removal-like interpretation is plausible because a non-head match is bypassed. A bit-to-tag association is also plausible because both operations use the selector. Neither inference establishes a domain-level meaning or a complete ownership policy.

## Evidence

The contract is based solely on the supplied static representation.

Established evidence includes the four argument widths, fixed-offset accesses, unsigned mask construction, full-width selector comparisons against one-byte tags, conditional bit clear, traversal and predecessor-link update, dependency call order, and final four-byte result store.

The supplied non-returning annotations support describing the match paths as terminal in the shown control flow, but do not establish the dependencies’ implementations.

No runtime tests have been performed. The acceptance examples express consequences of the supplied representation under their stated conditions; they are not observed executions.
