<!-- Trusted provenance: F-c20ae87ac99e; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-c20ae87ac99e

## Purpose

F-c20ae87ac99e accepts an input byte count, an input address, and a mutable context address. Context flags select one of two paths:

- **Transformed-block path:** directly converts input bytes into a context buffer and invokes F-fc23781a6360 after each completed 64-byte block.
- **Staging path:** may first complete an existing transformed block, then issue calls to an unresolved external dependency using addresses associated with a 4096-byte staging region.

“Block,” “staging,” and “occupancy” describe the visible organization and arithmetic. They do not establish an application protocol or the hidden purpose of any dependency.

In particular, the unresolved external dependency is **not established to copy bytes**. The evidence establishes its arguments and call ordering, but not its memory effects. Statements about actual copying require an explicit additional copying contract.

The function returns a 32-bit status. Its direct success result is zero. Other return sources are a nonzero result from F-d8a13f219e81 and a separately stored 32-bit fallback value. Earlier effects are not rolled back on failure.

## Interface

### Arguments and result

| Item | Representation | Established role |
|---|---|---|
| Requested count | Unsigned 32-bit integer | Number of input bytes requested |
| Input address | 32-bit address carried in an integer parameter | Base address for direct byte reads or source-like dependency arguments |
| Context address | 32-bit address carried in an integer parameter | Mutable context containing the transformed buffer and control fields |
| Result | Unsigned 32-bit integer | Zero, a forwarded prerequisite status, or the fallback value |

Direct byte operations use eight-bit values. Counts, address representations, and accessed scalar fields are 32 bits. Some staging calculations appear as signed 32-bit operations; their overflow limits are addressed under **Uncertainties**.

### Context members directly accessed

Offsets are in bytes from the context address.

| Offset | Direct access | Established use |
|---|---|---|
| 0–63 | Byte writes during ordinary block handling | Transformed-block buffer |
| 68 | 32-bit read at entry | Low six bits supply the initial buffer position |
| 92 | 32-bit read after a successful staging prerequisite | Address of an intermediary object |
| 100 | 32-bit read at entry | Path-selection flags |

The intermediary object supplies a 32-bit staging-object address at offset 4.

The staging object has these directly used members:

| Offset | Direct access | Established use |
|---|---|---|
| 76 | 32-bit read | Compared with staging-object address plus 84 |
| 80 | 32-bit write after each flush call | Set to zero |
| 104 | Repeated 32-bit reads and writes | Occupancy-like value |
| 108 | Address calculation | Base of the destination-like region passed to dependencies |

Under ordinary bounded occupancy, the nominal 4096-byte region beginning at offset 108 extends through offset 4203. This identifies an argument-address range, **not an established range of writes by the unresolved external dependency**.

These members do not constitute complete structure definitions.

### Preconditions and ownership

Executed direct accesses require valid readable or writable memory at the addresses used. Calls also require whatever memory and state their unknown contracts demand. The function performs no general validation of context validity, input extent, alignment, aliasing, or object lifetime.

Ordinary nonwrapping block handling requires readable input for the requested bytes. Staging does not itself directly read the remaining input bytes; whether a source-like argument must identify readable memory depends on the unresolved dependency’s contract.

No allocation, deallocation, or ownership transfer is explicit. Dependencies are not known to preserve input memory, globals, context fields, indirect objects, or object lifetimes.

## Behavior

### Entry reads and selection

The function first reads flags at context offset 100, then reads the value at offset 68. It retains only the low six bits of the latter, producing an initial position from 0 through 63.

The staging path is selected exactly when:

- The low eight flag bits equal 2.
- Bit `0x100` is set.

All other combinations select the transformed-block path. Bit `0x200` controls only whether the staging path first handles a preexisting partial block.

The entry flags and initial position are retained locally. Dependency mutations do not cause those entry fields to be reread for path selection.

### Direct byte transformation

Every byte directly written into the transformed buffer is calculated as:

**stored byte = (−99 × input byte − 20) modulo 256**

Input values 0, 1, and 255 therefore produce stored values 236, 137, and 79. Treating the loaded byte as signed rather than unsigned does not change these low-eight-bit results.

Each source byte is read before its corresponding destination byte is written. Processing follows increasing input index and destination position within each block. There is no input snapshot: overlapping writes can change later input reads.

Remaining input handled through staging receives no direct byte transformation from this function. That statement does not constrain transformations or other effects inside dependencies.

### Transformed-block path

The initial position and requested count are added using unsigned 32-bit arithmetic.

If the resulting sum exceeds 63, the function:

1. Converts enough bytes to fill positions from the initial position through 63.
2. Calls F-fc23781a6360 with the context address in both argument positions.
3. Repeats conversion of complete 64-byte blocks into positions 0–63 while the unsigned consumed-count-plus-64 expression is no greater than the original requested count.
4. Converts any suffix into the buffer beginning at position 0.

Every completed block is followed by its processing call before the next block or suffix is handled. An exact completion, including a total of exactly 64 bytes from the initial position, is processed immediately.

If the initial unsigned sum does not exceed 63, full-block handling is skipped. All requested bytes are converted starting at the initial position.

Finally, F-31d98408104a receives the context address and the original requested count. The function then returns zero. Neither dependency’s result is inspected on this path.

The function does not directly update the position field at offset 68. Any accounting or position change performed by F-31d98408104a remains outside the established direct behavior.

### Optional completion of a staged-path partial block

Before staging, an existing partial block is handled only if bit `0x200` is clear and the retained initial position is nonzero.

If the unsigned position-plus-count sum is below 64, all requested bytes are converted into the existing block. F-31d98408104a is called with the original requested count, and all requested input is treated as consumed.

Otherwise, enough bytes are converted to finish the block. The function then calls:

1. F-fc23781a6360 with the context address twice.
2. F-31d98408104a with the context address and the number of bytes just consumed.

Thus, accounting for this completed partial block precedes any staging prerequisite.

If the requested count is no greater than the consumed count, the function returns zero immediately. With ordinary arithmetic, this includes an exact partial-block completion.

If bit `0x200` is set, this handling is skipped even when the initial position is nonzero. The partial buffer is not directly cleared.

### Staging prerequisite and eligibility

When input remains, F-d8a13f219e81 is called with the context address and the integer 1.

- A zero result allows execution to continue.
- Every nonzero 32-bit result is returned unchanged.

Only after success does the function read the intermediary address from context offset 92. It computes the remaining count and advances the input address by the locally retained consumed count.

Eligibility requires:

1. A nonzero intermediary address.
2. A nonzero adjusted input address.
3. An initial staging occupancy below 4096.
4. The value at staging offset 76 equal to the staging-object address plus 84.

When either of the first two conditions fails, the intermediary member and staging members are not read. Otherwise, the function reads the staging-object address from intermediary offset 4, then reads its occupancy at offset 104. The offset-76 comparison is reached only if that occupancy is below 4096.

The staging-object address itself is not checked for zero before member access.

If eligibility fails, the function reads and returns the fallback value. Its value is unknown and is not established to be nonzero. This read occurs after any earlier prerequisite, block-processing, or accounting calls.

Once obtained, the staging-object address is retained. This function does not reacquire it from the context after subsequent dependency calls.

### Staging calls, flushes, and occupancy changes

For this contract, **F-ext-01** is a document-local opaque identifier for the unresolved external dependency. No supplied mapped identity or implementation is available for it.

At each staging-loop decision, the function reads occupancy from offset 104, adds it to the remaining count, and compares the unsigned interpretation of that 32-bit result against 4095.

If the result exceeds 4095, the following sequence occurs:

1. F-ext-01 is called with:
   - Destination-like address: staging-object address plus 108 plus the just-read occupancy.
   - Source-like address: current adjusted input address.
   - Length-like value: 4096 minus that occupancy.
2. F-d15c1be26a89 is called with the staging-object address, staging-object address plus 108, and 4096.
3. Zero is written to staging offset 80.
4. Occupancy is reread from staging offset 104.
5. Zero is written to staging offset 104.
6. The current input address is advanced, and the remaining count reduced, by **4096 minus the occupancy reread after the flush call**.

The first call’s arguments do not establish that any bytes were copied, read, or written. Likewise, “flush” identifies the placement of F-d15c1be26a89, not a proven output operation.

The post-flush occupancy reread is consequential. The amount used to advance input need not equal the earlier length-like argument. Either dependency may have changed occupancy before that reread.

When the loop condition is false:

1. Occupancy is read again to form a destination-like address at staging-object address plus 108 plus occupancy.
2. F-ext-01 is called with that address, the current input address, and the remaining count.
3. After the call, occupancy is reread.
4. The remaining count is added to that reread value and stored at offset 104.
5. The function returns zero.

The final invocation can have a zero length-like value. It is still invoked; no no-op behavior is established for that case.

No F-31d98408104a call is made specifically for the bytes represented by these staging calls. Results from F-ext-01 and F-d15c1be26a89 are not inspected.

### Arithmetic boundaries

Unsigned position-plus-count and consumed-count-plus-64 calculations can wrap at 32 bits. A wrapped initial sum can bypass full-block handling and cause the subsequent conversion loop to write beyond the nominal 64-byte buffer.

The staging loop compares an unsigned interpretation of a 32-bit sum. Negative-looking signed representations can therefore satisfy its greater-than-4095 condition.

No separate occupancy range validation is repeated after dependency calls. Mutations can consequently produce out-of-range address calculations, unexpected length-like arguments, or failure to make ordinary progress.

Some staging additions, subtractions, and address calculations are represented using signed 32-bit types. Exact behavior beyond their signed range is not established. The contract does not impose checked arithmetic, saturation, or a universal wrapping rule on those operations.

## Side effects

Established direct effects include:

- Reading context flags, initial position, and eligible indirect references.
- Reading and transforming input bytes on block-handling paths.
- Writing transformed bytes into context-relative addresses.
- Clearing staging offsets 80 and 104 after each flush call.
- Updating staging offset 104 after the final F-ext-01 call.
- Reading the fallback value when staging eligibility fails.

Calls expose the specified addresses and counts to dependencies. Their memory effects, external output, and ownership effects remain unknown. In particular, this function’s call to F-ext-01 does not establish that the staging region is populated.

There is no rollback. A partial block can be converted, processed, and accounted for before a later prerequisite failure or eligibility failure.

For a zero-byte request:

- The transformed-block path calls F-31d98408104a with zero.
- The staging path also calls F-31d98408104a with zero when the initial position is nonzero and bit `0x200` is clear.
- Other staging-path cases return zero without dependency calls.

Entry field reads still occur. No direct input-byte read or transformed-buffer byte write is needed in these zero-count cases.

## Dependencies

| Opaque ID | Established invocation | Unresolved behavior |
|---|---|---|
| F-fc23781a6360 | Context address twice after each completed transformed block | Processing, memory effects, ownership, and result meaning |
| F-31d98408104a | Context address and original or partial consumed count | Accounting semantics and all mutations |
| F-d8a13f219e81 | Context address and 1 before staging eligibility checks | Operation meaning and status meanings beyond zero/nonzero control flow |
| F-d15c1be26a89 | Staging object, region base at offset 108, and 4096 | Output behavior, occupancy changes, other mutations |
| F-ext-01 | Destination-like address, source-like address, and length-like value | Whether it copies anything; access extent, overlap handling, zero-length effects, failures, and ownership |

F-ext-01 is only a local opaque designation for the unresolved call target, not an asserted identity.

All described continuation behavior assumes dependencies return normally. The source establishes no recovery mechanism for dependencies that do not return normally.

## Acceptance examples

These are conditional consequences of static evidence, not executed tests. Unless stated otherwise, addresses are valid, relevant arithmetic stays within its established range, input does not overlap mutable state, and dependencies return normally without unmentioned mutations.

1. **Converted suffix.**  
   With transformed-block selection, initial position 0, and input bytes 0, 1, 255, context positions 0–2 receive 236, 137, 79. No F-fc23781a6360 call occurs. F-31d98408104a receives count 3; the result is zero.

2. **Exact transformed completion.**  
   With initial position 60 and count 4 on the transformed-block path, four converted bytes fill positions 60–63. F-fc23781a6360 is called once, followed by F-31d98408104a with count 4.

3. **Multiple transformed blocks.**  
   With initial position 0 and count 130, two block-processing calls occur. The last two input bytes are then converted into positions 0 and 1. Accounting receives 130 once.

4. **Partial block without staging.**  
   With staging selected, bit `0x200` clear, initial position 10, and count 5, five converted bytes occupy positions 10–14. Accounting receives 5. No prerequisite or indirect staging access occurs.

5. **Failure after partial-block effects.**  
   With staging selected, bit `0x200` clear, initial position 60, and count 10, four bytes complete the block. Processing and accounting occur before the prerequisite. If F-d8a13f219e81 returns 7, the function returns 7 without issuing staging calls for the remaining six bytes.

6. **Partial-buffer bypass.**  
   With staging selected, bit `0x200` set, a nonzero initial position, and a positive count, no partial-block conversion precedes the prerequisite.

7. **Exact staging-boundary call sequence.**  
   With eligible occupancy 4090 and six remaining bytes, F-ext-01 receives destination-like offset 4198, the current source-like address, and length-like value 6. F-d15c1be26a89 follows with length 4096. If occupancy remains 4090 through those calls, the function clears offsets 80 and 104, advances the source-like address by six, and makes a final F-ext-01 call with length-like value zero. If that final call leaves occupancy unchanged, the final stored occupancy is zero. No byte-copy effect is required by this criterion.

8. **Content consequence only under an added copying contract.**  
   Suppose occupancy is 10, three bytes remain, and F-ext-01 is additionally specified to copy exactly the requested bytes without other mutations. No flush occurs. Those three bytes then occupy staging offsets 118–120, and occupancy becomes 13. Without that additional contract, only the call arguments and final occupancy update are established—not those contents.

9. **Flush mutation changes progress.**  
   With occupancy 4090 and six remaining bytes, suppose the flush leaves occupancy zero. The subsequent advancement is 4096, not six, and the remaining signed value becomes −4090. If occupancy remains zero, its unsigned loop-comparison sum exceeds 4095, so another iteration is selected. Ordinary six-byte completion is not guaranteed.

10. **Ineligible initial occupancy.**  
    After prerequisite success, an initial occupancy of 4096 fails eligibility. The offset-76 comparison is skipped, no F-ext-01 call occurs, and the fallback value is returned.

## Uncertainties

The following remain **documented_only**:

- Application-level meaning of the flags, transformation, context, and identity comparison.
- Complete layouts, alignment constraints, and dependency preconditions.
- All dependency effects on globals, input, context, indirect objects, and ownership.
- Whether F-ext-01 performs copying at all, including zero-length and overlapping cases.
- Whether F-d15c1be26a89 emits data or preserves occupancy.
- The fallback value and whether it necessarily indicates failure.
- Exact semantics of represented signed-overflow cases.
- Outcomes after invalid accesses, dependency-induced invalidation, extreme counts, or unsuitable aliasing.

No additional validation, alternate staging implementation, retry policy, transaction guarantee, or ownership transfer is established.

## Evidence

The supplied static source establishes direct access offsets, entry-read order, branch conditions, byte transformation, dependency arguments, direct stores, return selection, and the occupancy rereads described above.

The buffer-role terminology is inferred intent supported by sizes, arithmetic, and call placement. Actual copying by F-ext-01 and actual output by the flush dependency are not established.

No runtime tests have been performed. The acceptance examples are conditional static criteria, not observations of execution.
