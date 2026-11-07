<!-- Trusted provenance: F-0d3bc7e98dd1; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-0d3bc7e98dd1

## Purpose

F-0d3bc7e98dd1 partitions an unsigned requested count into chunks using a selected boundary, a mutable context position, and a descriptor-associated endpoint. Each chunk is submitted to F-e11c387e41de. A nonzero submission result causes the function to advance a working value, increment a cached iteration value, and add the submitted size to the context position and a caller-supplied accumulator.

The established behavior is selection, dependency invocation, and bookkeeping. The units of the requested count, chunk size, position, endpoint, and boundary are not established. Describing these quantities as byte counts would require an additional contract.

The second argument is a 32-bit working value. The function forwards it to F-e11c387e41de and advances it arithmetically; it never dereferences it. A buffer-address interpretation is **inferred intent**, not an established interface requirement. It could instead represent an offset or another numeric cursor.

The call pattern for F-d7be9cfb4000 resembles a quotient-and-remainder operation. That interpretation is also **inferred intent**.

A zero function result does not establish complete fulfillment of the request. The cached iteration bound can terminate processing while the remaining count is nonzero.

This contract describes static evidence only. No runtime tests have been performed.

## Interface

The function accepts four arguments and returns a 32-bit result.

| Argument | Established role |
|---|---|
| Context address | Identifies storage containing a selector, descriptor-table address, descriptor index, and mutable position. |
| Initial working value | A 32-bit integer forwarded to F-e11c387e41de and advanced by each successfully submitted size. |
| Requested count | An unsigned 32-bit quantity reduced by each successfully submitted size. |
| Accumulator address | Identifies a 32-bit word incremented after successful submissions. The function does not initialize it. |

The supplied representation uses 32-bit address-bearing values for direct memory accesses. Behavior with wider pointers is not established.

The initial working value is not itself a direct-memory-access operand in this function. Its validity as an address, offset, handle, or other quantity depends on the unknown contract of F-e11c387e41de.

### Context and descriptor layout

The offsets and strides below describe memory layout in bytes; they do not establish byte units for the processed quantities.

| Context offset | Width | Interpretation in this function |
|---|---:|---|
| `0x110` | 16 bits | Signed selector. |
| `0x130` | 32 bits | Descriptor-table base address. |
| `0x138` | 32 bits | Descriptor index used with an eight-byte stride. |
| `0x13c` | 32 bits | Position-like word, read during setup and processing and updated after successful submissions. |

The function reads a 32-bit word at offset four within the selected descriptor. This contract calls that word the **endpoint**. Whether it denotes a length, an exclusive end, or another quantity is unknown.

The first four bytes of the selected descriptor are not directly accessed. Descriptor addressing uses signed 32-bit expressions in the supplied representation. Overflow in those expressions has no established portable behavior.

### Fixed boundary data

Two fixed 32-bit data locations participate in selection:

- **Location A:** supplies the boundary for selector −16371.
- **Location B:** supplies the boundary for selector −16368.

Their contents, initialization, and mutability are unknown. These are data locations, not callable dependencies. Some selection paths read values that are subsequently discarded.

### Results

| 32-bit representation | Signed interpretation | Established condition |
|---|---:|---|
| `0x00000000` | 0 | Selection was accepted and processing ended without a zero submission result. |
| `0xfffffff5` | −11 | The final selected boundary equals −1. |
| `0xfffffffd` | −3 | F-e11c387e41de returned zero for a submitted chunk. |

The function does not explicitly update a separate error-state location. Dependency effects on any such state remain unknown.

### Access requirements

Directly accessed storage must support the indicated operations:

- The selector must be readable.
- Selector −16372 requires neither fixed-data location.
- Every other selector requires location A to be readable.
- Every selector except −16372 and −16371 additionally requires location B to be readable.
- Accepted selection requires context-position and descriptor-related reads, even for a zero request.
- Successful-submission bookkeeping requires readable and writable position and accumulator words.

No explicit validation establishes null-address handling, alignment checks, descriptor bounds, valid index ranges, non-aliasing, or address-overflow handling. Invalid memory accesses have no established recoverable error result.

No direct access requirement can be assigned to the working value itself. Whether the submission dependency interprets it as an address and accesses associated storage is **documented_only**.

## Behavior

### Boundary selection

The first direct read obtains the signed 16-bit selector. Selection follows this table:

| Selector | Ordered fixed-data reads | Final selected boundary |
|---|---|---|
| −16372 (`0xc00c`) | None | 2000 |
| −16371 (`0xc00d`) | A | Value from A |
| −16368 (`0xc010`) | A, then B | Value from B |
| Any other selector | A, then B | −1 |

Location A is read on every path except selector −16372. Whenever B is read, its read follows A, regardless of A’s value.

Only the final selected boundary is checked against −1:

- Selector −16371 rejects if A contains −1, without reading B.
- Selector −16368 rejects if B contains −1, after reading A and B.
- For selector −16368, a discarded −1 from A does not cause rejection.
- Every unrecognized selector reads both locations, discards their values, and returns −11.

Rejection precedes both dependency calls and any direct position or accumulator write.

Zero and negative boundaries other than −1 are not explicitly rejected. Their consequences depend on dependency behavior and arithmetic conditions that are not fully established.

### Setup and cached values

Accepted selection causes two calls to F-d7be9cfb4000, in order:

1. The function reads the context position and supplies that 32-bit value with the selected boundary. The primary 32-bit result becomes the **current iteration value**.
2. After the first call returns, the function looks up the selected descriptor’s endpoint, subtracts one, and supplies that result with the selected boundary. The primary 32-bit result becomes the **terminal iteration value**.

The descriptor lookup for the second call occurs after the first dependency invocation. It can therefore observe changes that invocation made to the descriptor-table address, descriptor index, or descriptor contents.

Within a descriptor lookup, the relative evaluation order of the table-address and index reads is not independently established.

Both primary results are cached. Later position or descriptor changes do not cause these setup results to be recomputed. The working value initially equals the second argument.

### Eligibility and termination

An iteration is eligible only when both conditions hold:

- The current iteration value is unsigned-less-than the terminal iteration value plus one.
- The remaining requested count is nonzero.

The terminal-plus-one calculation is unsigned 32-bit arithmetic. A terminal result of `0xffffffff` therefore produces an upper bound of zero, making the comparison false for every current value. The condition cannot universally be replaced by “current is at most terminal.”

If processing is initially ineligible, the function returns zero after both setup calls. It makes no chunk submission and performs no direct bookkeeping write.

After successful iterations, processing ends when either eligibility condition becomes false. Exhaustion of the cached iteration range can leave an unfulfilled count.

### Per-iteration observations

Each eligible iteration first rereads the context position and invokes F-d7be9cfb4000 with that value and the current boundary.

The primary result is unused. A secondary 32-bit result becomes the iteration’s **offset-like value**. The supplied representation exposes that secondary result but does not establish its full calling convention.

When the cached current and terminal iteration values are equal, the function additionally:

- Looks up the endpoint again through the context’s then-current table-address and index fields.
- Calls F-d7be9cfb4000 with the endpoint itself, without subtracting one, and the current boundary.
- Replaces the boundary with this call’s secondary result if that result is nonzero.

A zero endpoint secondary result leaves the boundary unchanged.

The offset-like value remains the secondary result associated with the position call. It is not replaced by the endpoint call’s secondary result.

The terminal endpoint lookup follows the position dependency call and can observe that call’s memory effects. Neither call is established to preserve context fields, descriptor contents, or other accessible memory.

### Chunk selection

The available span is the current boundary minus the offset-like value. Its 32-bit representation is compared as unsigned with the remaining requested count.

The chunk size is the smaller of:

- The unsigned available-span representation.
- The unsigned remaining requested count.

The subtraction uses signed operands in the supplied representation. Cases requiring signed overflow remain **documented_only**; this contract does not assign them mathematical, saturating, or portable wrapping behavior.

No explicit positive-size check exists. A zero-sized chunk may be submitted. A negative subtraction result that does not overflow is compared through its unsigned 32-bit representation, rather than being explicitly rejected as a negative span.

### Submission and result handling

F-e11c387e41de receives five arguments in this order:

1. Context address.
2. Current working value.
3. Chunk size.
4. Offset-like value.
5. Current boundary.

A zero dependency result causes immediate return −3. The enclosing function performs no successful-submission bookkeeping for that attempt. The dependency may nevertheless have modified memory or caused other effects before returning zero.

Every nonzero result triggers bookkeeping for the entire submitted size. The result is not interpreted as a partial-completion count or as a signed success code.

### Successful-submission bookkeeping

After a nonzero submission result, these actions occur in order:

1. Add the chunk size to the working value.
2. Increment the cached current iteration value.
3. Reread the context position and write that value plus the chunk size.
4. Read the accumulator and write that value plus the chunk size.
5. Subtract the chunk size from the remaining requested count before the next eligibility test.

Working-value advancement is integer arithmetic. It is not established pointer arithmetic and does not itself establish any storage access.

The position addition uses the value read after F-e11c387e41de returns. It need not use the position previously supplied to F-d7be9cfb4000.

The position write precedes the accumulator read. If both designate the same word, the accumulator addition observes the already-updated position. Other aliases can affect later position, selector, descriptor, or accumulator observations.

A successful zero-sized submission still increments the iteration value and performs both bookkeeping reads and writes. With stable, non-aliased storage, those writes preserve the stored word values.

### Arithmetic widths

Iteration values, requested counts, chunk sizes, dependency results, working values, and bookkeeping words have 32-bit representations.

The following calculations use unsigned 32-bit arithmetic:

- Terminal-bound increment.
- Current-iteration increment.
- Remaining-count subtraction.
- Working-value advancement.
- Position and accumulator additions involving the unsigned chunk size.

Overflow in these unsigned calculations wraps modulo \(2^{32}\).

The supplied representation also contains signed context-address calculations, descriptor-address calculations, endpoint-minus-one, and boundary-minus-offset. Signed-overflow cases in these expressions remain **documented_only**. Unbounded arithmetic must not replace the observed widths.

## Side effects

Direct writes are limited to the context position and accumulator after nonzero submission results. The function does not directly initialize the accumulator, dereference the working value, allocate memory, or release memory.

Direct reads include discarded fixed-data values. An unrecognized selector reads A and then B before returning −11. A zero request does not bypass boundary selection, accepted-setup reads, or either setup dependency call.

The dependencies’ effects are not restricted by this direct-access inventory. Neither dependency is established to preserve the context, descriptor table, accumulator, fixed-data locations, globals, or other accessible memory.

There is no explicit rollback. Earlier successful submissions cause bookkeeping writes when execution reaches them. Their continued visibility after later dependency calls is not guaranteed: a later call, including a failing submission, may overwrite those words.

Likewise, recorded progress need not describe all effects of a dependency. A zero submission result does not establish failure atomicity.

No ownership acquisition, transfer, retention, or release is established. In particular, forwarding the working value does not prove that it identifies owned storage. If a dependency interprets it as a storage address, its access direction, retention, lifetime, alignment, and ownership requirements remain unknown.

## Dependencies

### F-d7be9cfb4000

Established usage:

- Receives two 32-bit arguments.
- Supplies primary results used as cached iteration values during setup.
- Supplies secondary results used as per-iteration offsets and possible boundary replacements.
- Is called twice during accepted setup.
- Is called once per eligible iteration, with one additional call when cached current equals cached terminal.

**Inferred intent:** the usage resembles division returning quotient and remainder. Under that interpretation, primary results identify partitions, position secondary results identify offsets within partitions, and endpoint secondary results adjust a terminal span.

That interpretation does not establish signedness, rounding, division-by-zero behavior, exceptional behavior, the secondary-result convention, memory preservation, or ownership effects.

### F-e11c387e41de

Established usage:

- Receives the five submission arguments listed above.
- A zero result causes return −3.
- Every nonzero result causes bookkeeping for the full submitted size.

The meaning of submission, the working value’s interpretation, quantity units, internal progress, memory access, failure atomicity, retention, ownership, and error-state effects remain **documented_only**.

## Acceptance examples

These are conditional consequences of static evidence, not executed tests. Unless stated otherwise, each assumes accessible storage, no relevant dependency mutation, no aliasing, and no unresolved signed overflow. All processing quantities are unit-neutral.

1. **Literal boundary.** Selector −16372 selects 2000 without reading A or B.

2. **First candidate rejects.** Selector −16371 with A equal to −1 reads A only and returns −11. Neither dependency is called.

3. **Discarded candidate.** Selector −16368 with A equal to −1 and B equal to 2000 reads both locations and accepts boundary 2000.

4. **Unrecognized selector.** Selector zero reads A and then B, discards both values, and returns −11 regardless of their contents.

5. **Zero request.** With selector −16372 and requested count zero, both setup calls still occur using boundary 2000. No chunk is submitted; the function returns zero.

6. **Request-limited submission.** An eligible nonterminal iteration with boundary 2000, offset 125, and remaining count 300 submits size 300. A nonzero result advances the working value by 300 and adds 300 to position and accumulator. The request is exhausted.

7. **Boundary-limited submission.** Boundary 2000, offset 1900, and remaining count 250 produce size 100. Success leaves a remaining count of 150. Further processing depends on the cached iteration bound.

8. **Terminal adjustment.** On a terminal iteration, position secondary result 100 and endpoint secondary result 450 produce boundary 450 and available span 350. An endpoint secondary result of zero instead retains the previous boundary.

9. **Later failure with preservation.** A size-100 submission succeeds, and the next submission returns zero. The result is −3. Under the preservation assumptions, the earlier increments remain visible. No bookkeeping occurs for the failed attempt.

10. **Later failure with mutation.** After an earlier successful submission, the next submission overwrites position and accumulator before returning zero. The function returns −3 without restoring the earlier bookkeeping values.

11. **Termination with an outstanding count.** If incrementing the current iteration makes the cached-bound comparison false while the remaining count is nonzero, the function returns zero with part of the request unprocessed.

12. **Wrapped upper bound.** Terminal primary result `0xffffffff` produces upper bound zero. No chunk is submitted, even for a nonzero request.

13. **Zero-sized success.** Equal boundary and offset values produce size zero. A nonzero submission result still increments the iteration value and performs position and accumulator bookkeeping.

14. **Aliased bookkeeping.** If position and accumulator designate the same word containing 100 immediately before bookkeeping, a successful size-20 submission first writes 120, then reads 120 and writes 140.

15. **Working-value wrap.** A working value of `0xfffffff0` advanced by a successful size-32 submission becomes `0x00000010`. If another iteration is eligible, that value is forwarded to the next submission. This arithmetic establishes no address validity or memory access.

## Uncertainties

The following remain **documented_only**:

- Units and semantic compatibility of counts, chunk sizes, spans, positions, endpoints, boundaries, and working-value increments.
- Whether the working value represents an address, offset, or another numeric cursor.
- Contents, initialization, and mutability of A and B.
- Exact semantics and secondary-result convention of F-d7be9cfb4000.
- Semantic meaning and valid ranges of position and endpoint.
- Dependency behavior for zero or negative boundaries.
- Signed-overflow behavior in the identified expressions.
- Behavior for inaccessible or misaligned storage.
- Caller aliasing, lifetime, and synchronization requirements.
- Dependency memory effects, ownership effects, and exceptional behavior.
- Preservation of bookkeeping across subsequent dependency calls.
- Whether callers interpret zero as complete fulfillment despite cached-bound termination.

These unknowns do not erase established sequencing, argument forwarding, arithmetic widths, or direct bookkeeping writes.

## Evidence

Established evidence consists of the supplied static representation: memory-access widths and offsets, selector-dependent reads, dependency arguments, primary and secondary result usage, unsigned eligibility comparison, chunk selection, return conditions, and bookkeeping order.

The representation establishes that A precedes B when both are read, accepted setup occurs even for a zero request, and successful submissions cause ordered position and accumulator updates. It also establishes that the second argument is forwarded and advanced without being directly dereferenced.

No dependency implementation or caller contract was supplied. Buffer-address interpretation, byte-oriented units, transfer semantics, and quotient/remainder semantics are not established. The latter interpretations remain inferences rather than requirements.

No runtime tests have been performed.
