<!-- Trusted provenance: F-35b8bd5fa9e0; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-35b8bd5fa9e0

## Purpose

F-35b8bd5fa9e0 selects a 32-bit unsigned result by comparing a value obtained from F-11c1b7e655b3 against a zero-terminated sequence of bytes.

The supplied representation establishes the following behavior for stable, valid input memory:

- The first byte is a return value associated with a successful match.
- Subsequent bytes form the sequence searched.
- A zero byte terminates the search and is not eligible to match.
- A matching nonzero byte causes the function to return the first byte, widened to a 32-bit unsigned value.
- Reaching the terminator without a match causes the function to return one if the first byte is zero, and zero otherwise.

This resembles a membership-controlled decision, but that interpretation is inferred intent. The established behavior is the byte search and return-value selection described below. In particular, a successful match does not necessarily return a Boolean value.

## Interface

### Inputs

The function receives two arguments:

1. **An opaque four-byte argument.**  
   The supplied body does not inspect this argument, use it in an address calculation, or explicitly pass it to F-11c1b7e655b3. No meaning for its contents is established.

2. **A pointer to byte-addressable memory.**  
   Each element is an unsigned eight-bit value. The function treats the memory as this logical layout:

   | Byte offset | Established role |
   |---|---|
   | 0 | Return selector used after a match or after reaching the terminator |
   | 1 and onward | Candidate bytes, ending at the first zero encountered by the search |

The pointer is not checked for nullness, accessibility, allocation size, or termination.

The byte at offset zero is not part of the searched sequence. It may itself be zero without terminating the search.

### Output

The result is a 32-bit unsigned integer.

For stable input memory:

- On a match, the result is the value at offset zero, in the range 0 through 255.
- On termination without a match, the result is either zero or one.

The comparison target is obtained from F-11c1b7e655b3 and is used as a full-width unsigned value. It is not explicitly reduced to eight bits. A target greater than 255 therefore cannot equal a candidate byte.

### Preconditions for defined completion

A normally completing invocation requires:

- F-11c1b7e655b3 to return normally.
- Every byte actually examined to be readable.
- Offset zero to be readable when the function reaches a return path.
- The search to encounter either a matching nonzero byte or a zero terminator before its signed indexing arithmetic exceeds its representable range.
- All address calculations involved in those accesses to be valid under the applicable execution model.

The function does not establish these conditions itself.

A terminator is not required before a successful match. Once a match occurs, later bytes are not examined.

## Behavior

### Dependency call precedes the search

The function calls F-11c1b7e655b3 once, with no explicit arguments shown.

This call occurs before any explicit access through the byte pointer in the supplied body. Its returned value becomes the comparison target.

The buffer contents relevant to the search are therefore the contents present after this dependency returns. If the dependency can modify that memory through some other reference, the function does not preserve a pre-call snapshot.

No conclusions are established about the dependency’s effects on globals, allocation state, ownership, or other externally visible state.

### Search begins at offset one

The search starts with the byte immediately following the return selector.

For each candidate position, the represented control flow gives the zero test priority:

1. The candidate byte is tested for zero.
2. If it is zero, the function takes the terminator return path without performing a target comparison for that position.
3. Otherwise, the candidate byte is compared with the full-width target.
4. Equality takes the match return path.
5. Inequality advances the search by one byte.

The byte at offset zero does not influence which candidate positions are examined.

For stable memory, the search stops at the earliest occurrence of either:

- A zero byte; or
- A nonzero byte equal to the target.

Bytes beyond that stopping point have no effect.

A target of zero never matches a stable candidate: a zero candidate takes the terminator path first. Similarly, a target outside the range 1 through 255 cannot match any stable candidate byte.

### Match return path

When a nonzero candidate equals the target, the function reads the byte at offset zero and returns its unsigned value widened to 32 bits.

It does not normalize that value to zero or one.

Consequently:

- A match with a selector of zero returns zero.
- A match with a selector of one returns one.
- A match with a selector of 200 returns 200.

A zero result alone does not distinguish a successful match from an unsuccessful search.

### Terminator return path

When the zero test detects a terminator, the represented body first derives a provisional unsigned result from one minus the byte at offset zero. It then tests the byte at offset zero against one. If that latter value is greater than one, the result is replaced with zero.

For stable memory, this has the following complete behavior:

| Selector at offset zero | Result after reaching the terminator |
|---|---:|
| 0 | 1 |
| 1 | 0 |
| 2 through 255 | 0 |

The subtraction uses integer promotion before conversion to the 32-bit unsigned result. Its mathematical range is −254 through 1, so the subtraction itself does not require signed overflow. Conversion of a negative provisional result to unsigned is modulo 2³². For a stable selector greater than one, the subsequent comparison replaces that provisional value with zero before return.

Thus the stable-memory fallback is a test for whether the selector is zero, not an unrestricted arithmetic complement.

### Read ordering and unstable memory

The supplied representation contains distinct source-level byte accesses:

- One candidate access for the zero test.
- A further candidate access for equality when the zero test did not take the terminator path.
- One selector access on the match path.
- Two selector accesses on the terminator path: one for subtraction and one for the greater-than-one test.

These accesses are not shown as volatile or atomic. This contract does not assert that every source-level occurrence necessarily produces a separate physical load, or that the function supports concurrent modification.

If distinct accesses can legally observe different values, the stable-memory simplifications do not necessarily hold. For example, the provisional fallback result may be based on one selector value while the decision to replace it with zero is based on another.

Likewise, if the candidate’s zero test and equality test observe different values, the value used for equality need not be the value that passed the zero test. No synchronized snapshot, atomicity, or concurrency guarantee is established.

## Side effects

### Direct effects

The supplied body contains:

- One explicit call to F-11c1b7e655b3.
- Reads of candidate bytes.
- Reads of the selector byte on a return path.
- Computation of the returned value.

It contains no explicit writes through the input pointer, no explicit global accesses, and no explicit allocation or release operation.

“No explicit writes” does not mean the invocation as a whole is free of side effects: the dependency’s behavior is unknown.

### Ownership and lifetime

The function does not explicitly take ownership of the buffer, free it, resize it, or retain its pointer.

That observation applies only to the supplied body. It does not establish that F-11c1b7e655b3 leaves the allocation or related global state unchanged.

The caller must provide storage that remains readable for all accesses actually reached. No independent lifetime protection is shown.

### Errors and abnormal completion

There is no explicit error result, bounds-failure result, exception handler, or recovery path.

An unreadable pointer, insufficiently accessible storage, invalid address calculation, or indexing overflow is not converted into a defined return value by this function.

If F-11c1b7e655b3 does not return normally, the byte search is not reached through the represented control flow.

## Dependencies

### F-11c1b7e655b3

**Established use:**

- Called exactly once on an ordinary invocation that enters the body and reaches the call.
- Receives no explicit arguments in the supplied representation.
- Runs before the function’s explicit buffer reads.
- Supplies the value compared against candidate bytes.
- Is not called again when a candidate fails to match.

**Required interpretation at this call site:**

Its returned value is held and compared as a 32-bit unsigned quantity. Candidate bytes are widened for comparison; the target is not explicitly truncated.

**Unknown behavior — documented_only:**

- Meaning and domain of the returned value.
- Whether the returned value is normally restricted to a byte-sized range.
- Global-state reads or writes.
- Buffer mutation through independently available references.
- Allocation, release, or ownership effects.
- Reentrancy and synchronization properties.
- Exceptional or non-returning behavior.

No preservation or ownership guarantee is attributed to this dependency.

## Acceptance examples

These are conditional acceptance cases derived from the supplied representation, not reports of executed tests. Except where stated otherwise, assume the dependency returns normally, memory is stable and readable, and indexing remains valid. Listed bytes begin at offset zero.

| Dependency result | Bytes | Expected result | Reason |
|---:|---|---:|---|
| 7 | 1, 7, 0 | 1 | The first candidate matches; return the selector. |
| 7 | 0, 7, 0 | 0 | A match returns the zero selector without normalization. |
| 7 | 200, 7, 0 | 200 | A match may return a non-Boolean value. |
| 7 | 0, 3, 0 | 1 | No match precedes the terminator; the selector is zero. |
| 7 | 1, 3, 0 | 0 | No match precedes the terminator; the selector is one. |
| 7 | 200, 3, 0 | 0 | A selector greater than one produces zero on the terminator path. |
| 7 | 0, 0, 7 | 1 | The immediate terminator stops the search before the later 7. |
| 0 | 0, 9, 0 | 1 | Zero is treated as a terminator, not as a match. |
| 256 | 0, 1, 255, 0 | 1 | No byte equals the full-width target 256. |
| 255 | 42, 255, 0 | 42 | The maximum byte value can match. |

Additional boundary conditions:

- **Selector excluded from search:** With target 7 and bytes 7, 3, 0, the result is zero. The selector’s value does not count as a match.
- **Early match limits access:** With target 7, a readable selector of 42, and a readable candidate of 7 at offset one, the result is 42 without requiring a later terminator.
- **No low-byte truncation:** A target of 263 does not match candidate 7, despite their equal low eight bits.
- **Missing stopping condition:** If neither a match nor a zero is reached within valid readable storage and representable indexing, no successful outcome is specified.
- **Dependency mutation:** If the dependency changes the buffer before returning, expected results must be derived from the subsequently observed bytes, not an assumed entry snapshot.

## Uncertainties

The following remain **documented_only**:

- The domain-specific meaning of the selector and candidate sequence.
- Why the opaque four-byte argument is present.
- Whether callers conventionally constrain the selector to zero or one.
- Whether callers guarantee termination or bounded sequence length.
- The target’s intended value range.
- Concrete calling-convention details beyond the represented input and result widths.
- Actual emitted load counts and optimization choices.
- Behavior under concurrent mutation, memory-mapped I/O, or other changing-memory conditions.
- Dependency effects not visible in the supplied body.
- Any higher-level distinction between a match returning zero and a fallback returning zero.

The search uses a signed 32-bit index in the supplied representation. Although candidate values are byte-sized, traversal is not thereby limited to 255 bytes. Conversely, arbitrarily long traversal is not established: evaluating the next candidate offset can exceed the signed index range. No wraparound policy or maximum supported buffer size is supplied.

## Evidence

**Established static evidence:** The supplied representation shows a four-byte opaque first argument, a byte-pointer second argument, a 32-bit unsigned result, one initial dependency call, forward traversal beginning at offset one, zero-before-equality precedence, and the two return paths described above.

**Inferred intent:** The layout appears to combine a selector with a zero-terminated membership sequence. The function may be intended as a predicate when callers restrict the selector to zero or one, but the body does not enforce that restriction.

**Verification status:** No runtime tests have been performed. Acceptance examples are conditional predictions from static evidence. No behavior of F-11c1b7e655b3 beyond its visible use at this call site has been assumed.
