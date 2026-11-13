<!-- Trusted provenance: F-d9cefd4f6311; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-d9cefd4f6311

## Purpose

The available static evidence describes a partially recovered routine that interleaves indirect calls with byte-sized and 32-bit memory writes. It derives addresses and stored values from its first and third inputs, incoming machine state, dependency results, and memory reads.

No application-level purpose is established. In particular, the evidence does not justify describing this routine as an allocator, serializer, object initializer, collection operation, or cleanup operation.

The visible path ends at an invalid-instruction boundary rather than an ordinary return. The contract below specifies the recovered ordering and data relationships, conditionally on execution reaching each operation. It does not establish that the entire path is executable.

## Interface

### Explicit inputs

| Input | Recovered width | Established use |
|---|---:|---|
| First input | 32 bits, unsigned | Passed to the first dependency; stored unchanged as a 32-bit value; logically shifted right by two for a later byte write. |
| Second input | 32 bits, uninterpreted | No explicit use appears in the recovered body. Its possible availability to indirectly called code through machine state is unknown. |
| Third input | 32 bits, uninterpreted | Stored once as a full 32-bit value and twice as its low eight bits. |

The absence of an explicit use of the second input is not evidence that changing it cannot affect execution.

### Implicit inputs

The visible operations also depend on:

- An incoming control-related machine value used to form the first indirect-call target.
- An incoming address-like machine value used for an early byte store.
- A secondary machine-result value later shifted right by three and passed onward.
- Results from numerous indirect calls, including results represented as two 32-bit halves of a 64-bit quantity.
- Existing bytes and 32-bit words at computed addresses.

These implicit inputs have no established validation or initialization contract.

### Output and termination

The recovered interface declares no return value. There is no visible normal return, success indicator, or error result.

The final recovered action represents an invalid-instruction termination boundary. Whether actual execution traps there, diverges earlier, transfers elsewhere, or cannot reach that boundary is **documented_only**.

### Numeric and memory model

This specification distinguishes:

- **Byte stores:** exactly the low eight bits of the stated value.
- **Word stores and reads:** 32 bits.
- **Logical right shifts:** zero-filling shifts of unsigned quantities.
- **Split results:** the low and high 32-bit halves of a recovered 64-bit dependency result.

Address calculations use the recovered 32-bit machine model. For bit-pattern examples, left shifts, additions, and multiplications are interpreted modulo \(2^{32}\). Several expressions were recovered with signed types, so these observations are not a portable signed-arithmetic guarantee.

Byte order, alignment requirements, memory mapping, and the validity of calculated addresses are unknown. A word store specifies its 32-bit value, not an established byte serialization.

## Behavior

### Reading the dependency notation

Opaque IDs below identify **call occurrences**, not proven distinct implementations. Different IDs may resolve to the same target. A range denotes consecutive calls in its stated order.

A call described as having “no displayed arguments” has no explicit arguments in the supplied representation; that does not prove that the callee receives no meaningful machine inputs. Dependency side effects remain unconstrained.

Every later step is conditional on earlier calls returning and earlier memory accesses completing.

### 1. Initial word and byte writes

The routine first retains the first input logically shifted right by two.

It invokes F-a001 with the first input, followed by F-a002 through F-a005 with no displayed arguments. Let the result of F-a005 be the **initial index**.

It then:

1. Stores the first input as a word at four times the initial index, plus offset 20.
2. Calls F-a006 with four times the initial index.
3. Calls F-a007 through F-a010 with no displayed arguments.

Let the result of F-a010 be the **first descriptor**. Before another dependency call, it:

- Stores the low byte of eight times the initial index at the incoming address-like value plus one.
- Stores the first descriptor shifted left by two as a word at eight times the initial index, plus offset 52.

### 2. Replacement of the descriptor word

F-b001 receives the first descriptor shifted left by two. F-b002 then produces an uninterpreted 32-bit result, which is passed to F-b003. F-b004 through F-b007 follow with no displayed arguments.

Let F-b007 supply a split result. The routine retains the first descriptor masked to its low 30 bits as the **descriptor base**.

It then:

- Writes the low byte of eight times the initial index at descriptor base plus one.
- Writes the high half of the split result at eight times the initial index, plus offset 52.

Thus the latter word location is explicitly written twice, with dependency calls between the writes.

F-b008 receives the low half of the split result shifted left by two. F-b009 through F-b012 follow. Their final result supplies the next call argument.

### 3. First pair of third-input byte writes

The routine now uses sixteen times the initial index as the **scaled initial base**.

Before F-c001, it writes:

- The low byte of the scaled initial base at address  
  \((\text{descriptor base} \mathbin{\text{logical-right-shift}} 20) + 17\).
- The low byte of the third input at scaled initial base plus 13.

F-c001 receives two identical arguments, each equal to the result of F-b012 shifted left by two. F-c002 returns a 32-bit value. F-c003 receives that value and its logical right shift by three. F-c004 through F-c007 follow.

The routine then writes:

- The low byte of the scaled initial base at absolute address 17.
- The low byte of the third input again at scaled initial base plus 13.

The repeated third-input store is not evidence of redundancy: the intervening dependencies may modify memory.

F-c008 receives the result of F-c007 shifted left by two. F-c009 through F-c012 follow.

### 4. Descriptor-base fields and a second scaled base

Let F-c012 return the **second index**. The routine writes:

- The complete third input at descriptor base plus 20.
- The low byte of twice the descriptor base at descriptor base plus 13.

It derives:

- A **second scaled base** equal to 32 times the second index.
- A **shifted second value** equal to the second index shifted left by seven.
- A **compact selector** equal to the second index masked to 27 bits, then logically shifted right by 17.

F-d001 through F-d003 are invoked with no displayed arguments. Let F-d003 return the **field value**.

The routine then stores:

- The shifted second value as a word at second scaled base plus 20.
- The low byte of the field value shifted left by two at second scaled base plus 13.

F-d004 receives the field value shifted left by two. F-d005 through F-d008 follow.

### 5. Selector-derived writes and a three-argument call

After F-d008, the routine writes the low byte of twice the retained right-shifted first input at compact selector plus seven.

It next reads a byte at second scaled base plus one. Using that byte as an unsigned address component, it writes the low byte of the compact selector at that byte’s value plus 17.

F-e001 receives three identical arguments, each equal to the field value shifted left by 21. F-e002 returns a new 32-bit value, the **retained control value**. F-e003 receives that value. F-e004 through F-e007 follow.

The byte at second scaled base plus one is then read **again**. This fresh value becomes the **captured selector byte**. The routine writes zero at captured selector byte plus 17.

The earlier byte read and this captured byte need not agree.

### 6. Captured selector state and low-address writes

F-f001 receives the result of F-e007 shifted left by two. F-f002 through F-f005 follow.

The routine writes:

- The low byte of the retained control value logically shifted right by three at second scaled base plus 13.
- The captured selector byte at absolute address 13.

F-f006 receives three arguments:

1. Twice the captured selector byte.
2. 152.
3. 608.

F-f007 follows with no displayed arguments. F-f008 receives a secondary machine-result value logically shifted right by three. The evidence does not establish that secondary value’s producing dependency or ABI meaning.

F-f009 through F-f012 follow. F-f013 receives the result of F-f012 shifted left by two and itself produces a result.

The routine then writes:

- The low byte of the retained control value logically shifted right by three at twice the captured selector byte, plus 13.
- The byte value 192 at absolute address 13.

The absolute-address write occurs second. If the captured selector byte is zero, both writes target address 13, and the second overwrites the first before the next call.

### 7. A word-derived address chain

F-g001 receives the result of F-f013 shifted left by two. F-g002 through F-g005 follow. Let F-g005 return the **link value**.

At twice the captured selector byte, plus 36, the routine stores the link value shifted left by two as a word. It then reads a word at twice the captured selector byte, plus 116. Call that read value the **loaded base**.

F-g006 receives, in order:

- The loaded base.
- The loaded base shifted left by one.
- The link value shifted left by five.
- The loaded base masked to 31 bits, then logically shifted right by two.

F-g007 through F-g010 follow. The routine then stores the link value shifted left by five at loaded base plus 36.

It next reads a word at loaded base plus 116 and passes that word to F-g011. F-g012 through F-g015 follow. F-g015 supplies another split result.

The loaded base is retained across these calls; it is not reloaded merely because dependencies may have modified the memory from which it originated.

### 8. Split-result-derived word writes

The routine stores the retained control value logically shifted right by three at four times the high half of F-g015’s result, plus 20.

F-h001 receives four arguments:

- The low half of F-g015’s result shifted left by two, repeated three times.
- Its high half logically shifted right by three.

F-h002 returns another split result. F-h003 receives its low half, high half, low half again, and high half logically shifted right by three.

F-h004 through F-h007 follow. Using the result of F-h007 as an index, the routine stores the low half of F-g015’s result shifted left by two at four times that index, plus 20.

F-h008 receives four times that index. F-h009 through F-h012 follow.

### 9. Final indexed-byte updates

Let F-h012 return the **final descriptor**. The routine reads a byte at loaded base plus one and writes the low byte of the final descriptor shifted left by two at the read byte’s value plus 17.

F-i001 through F-i003 follow. The routine then rereads the byte at loaded base plus one and writes, at that fresh byte’s value plus 17, the low byte of:

\[
(\text{final descriptor} \mathbin{\&} (2^{30}-1))
\mathbin{\text{logical-right-shift}} 20.
\]

F-i004 receives the result of F-i003 shifted left by two. F-i005 through F-i008 follow.

The routine reads a byte at the final descriptor shifted left by 12, plus one, and writes zero at that byte’s value plus 17.

F-i009 receives:

- The final descriptor shifted left by five, twice.
- The final descriptor shifted left by two.
- The result of F-i008 shifted left by two.

F-i010 through F-i013 follow. The routine again reads a byte at the final descriptor shifted left by 12, plus one, and writes zero at that fresh byte’s value plus 17.

F-i014 receives the result of F-i013 shifted left by two. F-i015 through F-i018 follow, after which recovered control flow ends at the invalid-instruction boundary.

## Side effects

### Memory access and ordering

The body contains explicit byte and word writes, byte-indexed writes, word reads, and writes to absolute addresses 13 and 17.

For byte-indexed writes whose destination is the read byte plus 17, the destination lies between addresses 17 and 272 inclusive. Such accesses must not be silently interpreted as offsets into an unspecified object.

Likewise:

- Descriptor base is between zero and \(2^{30}-1\).
- Compact selector is between zero and 1023.
- Compact selector plus seven is between addresses 7 and 1030.
- Twice the captured selector byte is between zero and 510.

These are numeric bounds, not evidence that the addresses are valid.

Explicit memory events and dependency calls occur in the order described above. Dependencies can observe preceding writes and can change memory used by subsequent reads. Repeated reads must therefore remain distinct.

Computed destinations may alias one another. No separation, object-size, or non-overlap guarantee is established.

### Ownership and lifecycle

No allocation, ownership transfer, retention policy, or release operation is established. A dependency may allocate, retain, mutate, release, or otherwise affect memory; none of those behaviors may be assumed absent.

The routine provides no visible rollback. A fault or non-returning dependency may leave earlier side effects in place.

### Preconditions and failures

Following the full visible path requires all executed call targets and memory accesses to be valid and all preceding dependencies to return. The supplied evidence does not establish those conditions.

There are no visible null checks, bounds checks, dependency-status checks, or recovery branches. This does not prove that dependencies perform no checks.

## Dependencies

All F-a001 through F-i018 identifiers used above are opaque occurrence identifiers assigned for this contract. They are not recovered symbols or established cross-reference identities.

The only supplied mapped function identity is F-d9cefd4f6311 itself. The actual identities, signatures, calling conventions, effects, and failure behavior of its indirect dependencies are unknown.

Targets are formed from an incoming control-related value or instruction-location-derived values with fixed additions. The evidence is insufficient to establish whether those calculations represent valid ordinary calls, unusual control-flow machinery, or recovery artifacts.

Neither absent displayed arguments nor an ignored displayed result establishes purity, lack of effects, or an ordinary no-argument interface.

## Acceptance examples

These are conditional static acceptance examples, not runtime results.

1. **First-input propagation.** If the first input is 19 and F-a005 returns 3, the initial word write stores 19 at address 32. Later, the compact-selector-plus-seven byte write stores 8, because the retained unsigned right shift produces 4 and doubling produces 8.

2. **Third-input truncation.** For third input \(0x123456AB\), the two scaled-initial-base-plus-13 writes each store byte \(0xAB\). The descriptor-base-plus-20 write stores the full word \(0x123456AB\).

3. **Overwritten descriptor field.** If the initial index is 2, both early descriptor-field word writes target address 68. The first stores the first descriptor shifted left by two; the second stores the high half returned by F-b007. Calls between them prevent treating them as one observable write.

4. **Fresh selector read.** If second scaled base plus one initially contains 5, the first indexed write targets address 22. If intervening dependencies change that byte to 9, the later zero write targets address 26, and 9 becomes the captured selector byte.

5. **Aliasing at address 13.** If the captured selector byte is zero, the late shifted-control byte write targets address 13. The immediately following explicit write replaces it with 192. Both writes remain part of the sequence.

6. **Shift truncation.** Under the recovered 32-bit model, a final descriptor of \(0x40000000\) shifted left by two becomes zero. Its later shift-left-by-12 byte-read address is therefore one, not an unbounded mathematical address.

7. **Interrupted execution.** If a dependency fails to return, no subsequent body operation is required by this recovered path. Prior writes are not specified to be undone.

## Uncertainties

The following remain **documented_only**:

- Reachability and correctness of the recovered control-flow sequence.
- Actual termination behavior at the invalid-instruction boundary.
- Dependency target identities and complete argument/result conventions.
- Meaning and origin of incoming machine state and the secondary result value.
- Whether split results genuinely form 64-bit application values.
- Memory mapping, alignment, byte order, volatility, and device-register behavior.
- Allocation boundaries, ownership, and dependency lifetime effects.
- Concurrency, synchronization, and reentrancy.
- Any application-level interpretation of the offsets or low-address stores.

An inferred intent is that some values act as scaled indices, compact selectors, or address-like descriptors. These labels describe numeric relationships only; they do not establish data structures.

## Evidence

Established static evidence consists solely of the supplied recovered representation: explicit inputs, typed-width accesses, masks and shifts, indirect-call ordering, result uses, repeated reads and writes, and the invalid-instruction boundary.

The behavioral ordering above is conditional on that representation being faithful. It is not validated machine execution.

No runtime tests, dependency inspection, memory-layout inspection, or implementation verification have been performed.
