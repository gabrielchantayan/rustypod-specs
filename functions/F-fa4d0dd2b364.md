<!-- Trusted provenance: F-fa4d0dd2b364; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-fa4d0dd2b364

## Purpose

F-fa4d0dd2b364 prepares a caller-supplied buffer with fixed fields and a trailing pair of 16-bit fields, then repeatedly submits that buffer through F-3d3ba69a50c8. Before each submission, it supplies a per-iteration value to F-946cac4b59bc.

The established behavior is buffer initialization, ordered field writes, dependency calls, position advancement, and propagation of the first nonzero submission status. The apparent intent is to submit a sequence of equally sized units, with a zero value marking the final iteration. That interpretation is inferred; the supplied evidence does not establish a storage format, protocol, device type, or meaning for the field constants.

The requested iteration count has an important boundary: the loop’s index update clears bit 16. Consequently, counts above 65,535 do not produce ordinary count-bounded completion. They repeat an index cycle unless a submission returns a nonzero status.

## Interface

The function accepts six 32-bit arguments, described here by their roles:

| Argument | Width and interpretation | Established use |
|---|---|---|
| Context | Opaque 32-bit value | Passed unchanged to F-3d3ba69a50c8. |
| Initial position | Unsigned 32-bit integer | Supplies the first submission position and the upper result word when there are no submissions. |
| Cursor | 32-bit address-like value | Advanced by one before each call to F-b5c826d2ab04. Not directly dereferenced by this function. |
| Requested count | Unsigned 32-bit integer | Controls the loop comparison and selection between a derived value and a final zero value. |
| Buffer size | Unsigned 32-bit integer | Passed to the initializer and submission dependency; also used for footer placement and position advancement. |
| Buffer address | 32-bit address-like value | Destination for direct writes and an argument to the initializer, F-946cac4b59bc, and F-3d3ba69a50c8. |

The result is a 64-bit value comprising two 32-bit words:

- **Lower word:** zero if the loop completes normally, otherwise the first nonzero lower-word status returned by F-3d3ba69a50c8.
- **Upper word:** the initial position if no submission is attempted; the buffer size if at least one submission is attempted.

The upper word is therefore not a consistent position, byte count, or number of completed iterations. Its interpretation depends on whether a submission was attempted.

The function has no explicit output-length parameter separate from the buffer size, no explicit allocation result, and no distinct local validation error.

### Memory layout

The function directly writes the following fields relative to the buffer address:

| Relative offset | Width | Value supplied to the store |
|---|---:|---|
| 8 | 8 bits | 2 |
| 10 | 16 bits | Result of F-0ec8efa9c769 with input 1 |
| Buffer size minus 2 | 16 bits | Result of F-0ec8efa9c769 with input 14 |
| Buffer size minus 4 | 16 bits | Result of F-0ec8efa9c769 with input equal to the low 16 bits of buffer size minus 6 |

The 16-bit stores use the target’s native representation. The evidence does not establish their byte order independently of F-0ec8efa9c769.

## Behavior

### Initialization and direct writes

The following sequence occurs before any loop-count check:

1. The unmapped external initializer is called with the buffer address and buffer size.
2. The byte at buffer offset 8 is written with value 2.
3. F-0ec8efa9c769 is called with input 1. Its 16-bit result is written at buffer offset 10.
4. F-0ec8efa9c769 is called with input 14. Its 16-bit result is written at buffer offset buffer size minus 2.
5. F-0ec8efa9c769 is called with the low 16 bits of buffer size minus 6. Its 16-bit result is written at buffer offset buffer size minus 4.

The final conversion input is a modulo-65,536 quantity. It is not a checked subtraction, a saturated length, or the full 32-bit size difference.

These operations occur even when the requested count is zero. Zero iterations do not imply an untouched buffer or absence of dependency calls.

The direct stores are interleaved with calls to F-0ec8efa9c769. Their order is established, but the final contents cannot be inferred solely from these stores if that dependency changes relevant memory or global state.

### Per-iteration value selection

The loop index starts at zero.

For an iteration whose index plus one is less than the requested count:

- The cursor advances by one before the dependency call.
- F-b5c826d2ab04 receives the advanced cursor and a 32-bit carried value.
- Its 32-bit result becomes the value supplied to F-946cac4b59bc.

For an iteration whose index plus one is not less than the requested count:

- The cursor is not advanced.
- F-b5c826d2ab04 is not called.
- The value supplied to F-946cac4b59bc is zero.

Thus, for normally completing positive counts up to 65,535, the first lookup uses one address unit beyond the original cursor. The original cursor itself is not passed to F-b5c826d2ab04 by this function.

The carried value has two origins:

- On the first iteration, it comes from secondary-register state associated with the final initialization conversion call.
- On later iterations, it is the upper 32-bit word of the preceding submission result.

The initial secondary-register value is not defined by the displayed 16-bit return interface of F-0ec8efa9c769. It must not be silently treated as zero or as a copy of that function’s primary result.

### Buffer update and submission

Each iteration proceeds in this order:

1. Select the derived value or zero as described above.
2. Call F-946cac4b59bc with that value and the buffer address.
3. Set the eventual upper result word to the buffer size.
4. Call F-3d3ba69a50c8 with six arguments:
   - the original context;
   - secondary-register state associated with the preceding F-946cac4b59bc call;
   - the current position;
   - a literal zero;
   - the buffer size;
   - the buffer address.
5. Split the submission result into two 32-bit words.
6. Preserve its upper word as the carried value for a possible later lookup.
7. Examine its lower word as the submission status.

The second submission argument is not established as a defined return value by the visible interface of F-946cac4b59bc. Its concrete value and significance are documented_only.

If the submission status is nonzero, the function returns immediately. It performs no position increment for that failed submission, makes no further loop calls, and does not locally restore the buffer or earlier state.

If the submission status is zero, the current position advances by the buffer size shifted right by nine bits. For an unsigned size, this is the integer quotient of the size divided by 512:

- Sizes below 512 advance the position by zero.
- Nonmultiples of 512 advance by the truncated quotient, not a rounded-up amount.
- Position addition is unsigned 32-bit arithmetic and wraps modulo \(2^{32}\).

The advance occurs after every successful submission, including the last successful submission of a normally completing loop. The resulting position is not returned.

### Loop update and count boundary

After a successful submission and position advance, the loop index is incremented and bit 16 is cleared. This produces a repeating index range from 0 through 65,535.

Consequences:

- **Requested count 0:** no iterations.
- **Requested counts 1 through 65,535:** exactly that many iterations, provided every submission returns zero.
- **Requested count 65,536:** the index never reaches the requested count. At index 65,535, the per-iteration value is zero; the index then returns to zero.
- **Requested counts greater than 65,536:** the index never reaches the requested count, and the zero-value branch is never selected. Every iteration advances the cursor and calls F-b5c826d2ab04.

For every count above 65,535, a nonzero submission status still terminates the function. Without such a status, the local loop has no normal completion path. Address exhaustion, dependency failure, or environmental interruption is not equivalent to a defined local return.

## Side effects

### Buffer and external state

Established direct memory effects are limited to the four stores listed in the interface. Additional effects may arise from:

- The initial external call.
- F-0ec8efa9c769.
- F-b5c826d2ab04.
- F-946cac4b59bc.
- F-3d3ba69a50c8.

The function does not directly read buffer contents or cursor-addressed contents. Any such reads occur within dependencies and are not specified by this evidence.

The fixed fields and footer are prepared once, not before every submission. If a dependency changes them, this function does not explicitly reconstruct them on the next iteration.

### Ownership and lifetime

The function contains no explicit allocation or release operation. That does not establish that its dependencies neither allocate nor release memory.

The buffer must be usable for the direct stores and all dependency operations that actually occur. Whether a dependency retains its address beyond a call, transfers ownership, or requires a longer lifetime is unknown.

The cursor’s required readable extent, alignment, and interpretation depend on F-b5c826d2ab04. This function alone does not establish those requirements.

### Bounds, overlap, and failures

There are no local checks for null addresses, available allocation size, alignment, footer overlap, or arithmetic overflow in address formation.

For an allocation exactly as large as the declared buffer size, the four explicit stores fit within that allocation when the size is at least 12, assuming ordinary nonwrapping address arithmetic and valid alignment. Sizes 12 through 15 still allow footer fields to overlap fixed fields. A size of at least 16 keeps the listed fixed fields and footer fields disjoint.

These are conditions for understanding the direct accesses, not proof that any size is valid for the dependencies.

Small sizes can place footer stores before the buffer or outside the declared region. The low-16-bit conversion input still follows its arithmetic rule; no local validation converts these cases into a controlled error.

The only explicit status-based early exit is a nonzero lower word from F-3d3ba69a50c8. Initialization and intermediate dependency calls have no visible status checks. Faults or failures inside them have no specified local recovery.

## Dependencies

| Opaque dependency | Established relationship |
|---|---|
| F-0ec8efa9c769 | Called three times with inputs 1, 14, and the low 16 bits of buffer size minus 6. Each primary result supplies a 16-bit store. Secondary-register state from the last call is later used as an argument. |
| F-b5c826d2ab04 | Called only on the non-final-value path, after advancing the cursor. Receives that cursor and a carried 32-bit value; produces the value passed onward. |
| F-946cac4b59bc | Called once per attempted submission with the selected 32-bit value and buffer address. Secondary-register state following this call supplies an argument to the submission dependency. |
| F-3d3ba69a50c8 | Called once per iteration with the six arguments described above. Its lower result word controls termination; its upper result word can feed the next lookup. |

An additional external initializer is present, but no opaque function ID was supplied for it. Only its position in the call sequence and its buffer-address and buffer-size arguments are established. Its name is not evidence that it clears, fills, allocates, or otherwise initializes memory in any particular way.

No dependency is assumed to preserve globals, leave unrelated memory unchanged, or preserve ownership.

## Acceptance examples

These are conditional static acceptance examples, not executed tests. Unless otherwise stated, all relevant memory accesses and dependency calls are assumed valid.

1. **Zero requested count**

   The initializer and all three conversion calls occur, together with the four direct stores. No per-iteration dependency is called. The result has lower word zero and upper word equal to the initial position.

2. **One successful submission**

   With requested count 1, the cursor is not advanced and F-b5c826d2ab04 is not called. F-946cac4b59bc receives zero and the buffer address. One submission occurs at the initial position. If its lower result word is zero, the function returns lower word zero and upper word equal to the buffer size.

3. **Two successful submissions with a 1,024-byte buffer**

   The first iteration advances the cursor once and calls F-b5c826d2ab04. The second iteration uses zero instead of another lookup. Submission positions are the initial position and the initial position plus 2, modulo \(2^{32}\). The returned upper word is 1,024.

4. **Immediate submission failure**

   If the first submission returns lower word 7, the function returns lower word 7 and upper word equal to the buffer size. It does not advance the position after that submission. Initialization and the first F-946cac4b59bc call have already occurred.

5. **Upper-word carry between iterations**

   With requested count at least 3, suppose the first submission returns status zero and upper word \(H\). The second F-b5c826d2ab04 call receives \(H\) as its second argument. The first lookup’s corresponding argument remains dependent on the initialization call’s secondary-register state.

6. **Sub-512 size**

   With a valid 511-byte buffer and two successful iterations, both submissions use the same position because the position increment is zero. The final conversion receives 505.

7. **Overlapping fields**

   With buffer size 12, the byte at offset 8 and the halfword at offset 10 are written before the footer halfwords at offsets 10 and 8. If intervening dependencies do not alter those locations, the later footer stores determine their final bytes. The function does not prevent this overlap.

8. **Count boundary**

   A requested count of 65,535 completes after 65,535 successful submissions. A requested count of 65,536 does not complete merely because that many submissions succeeded: its next index is zero. At count 65,537, every iteration takes the lookup path until an external condition stops execution or a submission returns nonzero.

## Uncertainties

The following remain **documented_only**:

- The initializer’s effects and failure behavior.
- The semantic meaning of all encoded fields and per-iteration values.
- Whether F-0ec8efa9c769 performs byte-order conversion or another transformation.
- The defined ABI meaning, if any, of the secondary-register values used as arguments.
- Cursor element format, dependency read widths, and required extent.
- Dependency effects on globals, buffer contents, ownership, and retained references.
- Alignment requirements and behavior for invalid addresses or overlapping accesses.
- Whether address-like arithmetic outside the valid address domain wraps, faults, or has another target-specific consequence.
- The signed or semantic interpretation of nonzero submission statuses.
- Whether the masked index update is intentional.

The loop-boundary behavior and mixed-purpose upper result word are established behavior, not corrected versions of inferred intent.

## Evidence

This contract is based solely on the supplied static function body and opaque dependency mapping.

The static evidence establishes argument use, explicit store widths and ordering, dependency call order, arithmetic masks and shifts, loop transitions, submission-status handling, and result-word construction. It does not establish dependency internals or environmental behavior.

No runtime tests have been performed. The acceptance examples are consequences of the visible control flow under their stated conditions, not observed execution results.
