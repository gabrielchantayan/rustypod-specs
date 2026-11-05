<!-- Trusted provenance: F-ef0c767f14fe; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-ef0c767f14fe

## Purpose

F-ef0c767f14fe compares two addressed records and returns one of two unsigned 32-bit values: **0x00000001** or **0xFFFFFFFF**.

For readable records whose relevant contents remain stable during the call, the comparison priorities are:

1. An unsigned primary high word.
2. An unsigned primary low word.
3. An unsigned secondary word.
4. The numerical unsigned record address.

At the first unequal component, a greater first-record component produces **0x00000001**, and a smaller first-record component produces **0xFFFFFFFF**. If all compared words and both addresses are equal, the result is **0x00000001**.

The function therefore never reports equality through a zero result. Stable self-comparison returns **0x00000001**. This is not the equality behavior of a conventional three-way comparator.

The unsigned comparison behavior is established by the supplied body. Treating the primary high and low words as one logical 64-bit key is a useful interpretation, not an independently established record format or domain meaning.

## Interface

### Inputs and record layout

The function receives two unsigned address-valued arguments, designated here as the **first address** and **second address**. Each identifies storage with the following relevant fields:

| Byte offset from address | Width in the recovered model | Contract name | Comparison priority |
|---|---:|---|---:|
| +0x0C | 32 bits | Secondary word | Third |
| +0x10 | 32 bits | Primary low word | Second |
| +0x14 | 32 bits | Primary high word | First |

The supplied representation is interpreted using 32-bit unsigned words and 32-bit unsigned address arithmetic. The original ABI and type definitions are not independently supplied.

Each field occupies four bytes in this model. No bytes outside offsets +0x0C through +0x17 are explicitly accessed. No meaning is established for other bytes in the record.

All field comparisons are unsigned. For example, **0x80000000** is greater than **0x7FFFFFFF**, and **0xFFFFFFFF** is greater than zero.

The arguments need not identify distinct records. The body contains no non-aliasing check or restriction. Identical addresses are handled by the same comparison logic as distinct addresses. Other forms of overlapping storage are not explicitly rejected; the relevant requirements are that every executed access is valid and, for the simplified ordering contract, the observed fields remain stable.

### Return value

| Unsigned 32-bit result | Meaning for stable records |
|---|---|
| 0x00000001 | The first record is greater under the stated ordering, or all components including the addresses are equal |
| 0xFFFFFFFF | The first record is smaller under the stated ordering |

The second result has the bit pattern commonly interpreted as signed negative one, but the supplied return type is unsigned. Both results are greater than zero when interpreted as unsigned values. Consumers cannot distinguish the outcomes by testing whether the unsigned result is positive.

There is no separate success status, equality status, invalid-input result, or error object.

### Preconditions

For an execution with valid memory accesses:

- Every field location actually read must support an unsigned 32-bit read under the target’s memory-access rules.
- Address-plus-offset calculations must identify accessible storage.
- Accessed storage must remain alive for the duration of those accesses.
- Any applicable alignment and pointer-representation requirements must be satisfied.

To obtain the stable-record ordering, the relevant words must additionally remain unchanged throughout the call.

The body performs no explicit null, bounds, alignment, lifetime, or address-overflow validation. A null or otherwise invalid argument is not handled through a dedicated return value. The platform’s response to an invalid access is undocumented.

Receiving addresses does not establish who owns the underlying storage.

## Behavior

### Stable-record ordering

For stable records, the primary high words decide the result when they differ. If they are equal, the primary low words decide when they differ. If both primary words are equal, the secondary words decide when they differ. Only when all three words are equal does the numerical address comparison determine the result.

The final address comparison is inclusive: the first address being greater than **or equal to** the second produces **0x00000001**.

Consequences include:

- A high-word difference dominates all low-word, secondary-word, and address differences.
- A low-word difference dominates secondary-word and address differences.
- A secondary-word difference dominates the addresses.
- Distinct records with identical compared contents are ordered by address.
- Identical addresses with stable contents produce **0x00000001**.
- No normal path returns zero.

The primary ordering is equivalent, for stable inputs, to comparing an unsigned quantity formed by multiplying the high word by 2³² and adding the low word. The body does not perform that construction: it uses separate word comparisons, without a combined 64-bit load, subtraction, or carry calculation.

### Primary admission stage

The first stage observes whether the second primary high word is less than or equal to the first primary high word.

It then makes a separate observation of high-word equality. If that equality observation succeeds, a comparison of the primary low words replaces the earlier decision: the second low word must be less than or equal to the first low word for the stage to pass.

A failed stage returns **0xFFFFFFFF** immediately. No secondary word is accessed on that path.

The initial high-word comparison and the following equality comparison are distinct source-level reads. Their observations are not guaranteed to agree if storage changes. In particular, low-word comparison depends on the later equality observation, not on an assumed snapshot shared with the first comparison.

### Primary reassessment stage

After admission, the body captures the first primary low word and then the first primary high word. These reads occur in that order.

Using those captured first-record values, the body makes separate observations of:

- Whether the second high word is less than or equal to the captured first high word.
- Whether another read of the second high word equals the captured first high word.
- If that equality observation succeeds, whether the second low word is less than or equal to the captured first low word. This replaces the high-word less-than-or-equal decision.
- Whether a further read of the second low word equals the captured first low word.

The last low-word equality observation is made even when the high-word equality observation failed.

The reassessment returns **0x00000001** without secondary reads when its less-than-or-equal decision is true and at least one of its high-word or low-word equality observations is false.

Otherwise, comparison continues using the original record addresses. For stable records that passed admission, this continuation occurs exactly when both primary words are equal. If observations change between reads, continuation can also occur because the reassessment’s less-than-or-equal decision is false.

Thus, stable ordering does not imply that every lower-priority primary field is skipped after a high-word difference. A stable greater first high word passes admission and reaches reassessment, where primary low words are still read. The secondary words, however, remain unread when stable primary keys differ.

### Secondary comparison and address decision

The secondary stage first observes whether the second secondary word is less than or equal to the first secondary word.

- If false, the function returns **0xFFFFFFFF**. It does not perform the later strict secondary comparison or the address comparison.
- If true, the body reads the secondary words again to test whether the second is strictly less than the first.
- If the strict comparison succeeds, the function returns **0x00000001** without evaluating the address tie-break.
- Otherwise, the numerical address comparison determines the result: **0x00000001** when the second address is less than or equal to the first, and **0xFFFFFFFF** otherwise.

For stable records, reaching the address comparison establishes equality of the secondary words. With changing observations, that conclusion is not justified: a successful earlier less-than-or-equal observation and an unsuccessful later strict-less observation need not describe one common state.

### Observation and arithmetic limits

The supplied control flow establishes the stages, conditional accesses, and explicit capture order described above. It does not establish which operand is read first within a comparison that reads both records.

Repeated source-level field accesses must not be treated as a single captured record snapshot. Conversely, the supplied representation does not establish an exact machine-load count, compiler optimization policy, volatile behavior, atomicity, or memory-ordering guarantee.

The comparison logic does not increment, decrement, or subtract field values. Zero and **0xFFFFFFFF** are ordinary unsigned boundaries, with no key-arithmetic overflow involved.

Field addresses are formed by adding fixed byte offsets. Under the recovered 32-bit unsigned model, those additions can wrap modulo 2³². The body neither detects nor rejects wrapping. This does not establish that wrapped accesses are valid or portable; validity of every executed access remains a precondition.

## Side effects

The supplied body explicitly performs record-field reads and local comparison-state changes, then returns one of the two specified values.

It contains no explicit:

- Memory writes.
- Allocation or deallocation.
- Ownership transfer or lifetime operation.
- Global-variable access.
- Callback or other function call.
- Synchronization or error-reporting operation.

**Ownership of the records is unknown.** The addresses may designate caller-owned storage or storage owned elsewhere; the supplied body does not distinguish those cases. Its absence of ownership or lifetime operations does not establish caller ownership.

Ordinary readable memory is not modified by any explicit operation in this body. Effects of reads from device-backed or otherwise special storage are undocumented. Faults, exceptions, signals, traps, or other platform consequences of invalid accesses are likewise undocumented.

## Dependencies

No function calls appear in the supplied body. There are therefore no callable dependency IDs to specify.

Execution relies on the target environment’s addressed unsigned-word reads, unsigned comparisons, and address representation. No allocator, external comparator, synchronization service, or error handler is explicitly invoked.

No claim about another function preserving globals, transferring ownership, or releasing memory is needed or supported.

## Acceptance examples

These are conditional acceptance criteria derived from static inspection, not observed runtime results. Unless stated otherwise, all executed accesses are valid, the records remain stable, and the addresses are distinct.

| First primary high:low | Second primary high:low | Secondary values | Address relation | Expected return |
|---|---|---|---|---|
| 0x00000002:0x00000000 | 0x00000001:0xFFFFFFFF | Any | Either order | 0x00000001 |
| 0x00000001:0xFFFFFFFF | 0x00000002:0x00000000 | Any | Either order | 0xFFFFFFFF |
| 0x00000003:0x00000008 | 0x00000003:0x00000007 | Any | Either order | 0x00000001 |
| 0x00000003:0x00000007 | 0x00000003:0x00000008 | Any | Either order | 0xFFFFFFFF |
| Equal | Equal | First 10; second 9 | Either order | 0x00000001 |
| Equal | Equal | First 9; second 10 | Either order | 0xFFFFFFFF |
| Equal | Equal | Equal | First address greater | 0x00000001 |
| Equal | Equal | Equal | First address smaller | 0xFFFFFFFF |

Additional criteria:

- **Unsigned primary boundary:** A first high word of **0x80000000** outranks a second high word of **0x7FFFFFFF**, regardless of low words, secondary words, or addresses.
- **Unsigned low-word boundary:** With equal high words, a first low word of **0xFFFFFFFF** outranks a second low word of zero.
- **Unsigned secondary boundary:** With equal primary keys, a first secondary word of **0xFFFFFFFF** outranks a second secondary word of zero.
- **Self-comparison:** Identical valid addresses with stable storage return **0x00000001**, not zero.
- **Admission rejection:** If stable first and second high words are 1 and 2 respectively, the function returns **0xFFFFFFFF** without reading primary low words or secondary words.
- **Greater-primary access behavior:** If stable first and second high words are 2 and 1 respectively, primary reassessment reads primary low words, but neither secondary word is read.
- **Content equality:** Equal compared words at distinct addresses do not produce an equality result; numerical address order decides.
- **No ownership implication:** The behavioral result does not depend on an ownership classification. No explicit ownership transfer or lifetime operation occurs.

A changing-observation example illustrates the limits of entry-state reasoning. Suppose the first high word remains 5. The second high word is observed as 4 during both admission comparisons, then as 6 during both reassessment high-word comparisons. Admission passes, but reassessment’s less-than-or-equal decision is false, so execution reaches the secondary comparison. If the first secondary word remains 9 and the second remains 10, the result is **0xFFFFFFFF**, despite the earlier primary observations favoring the first record.

That example is conditional on those observations being possible in the environment. It does not establish that concurrent unsynchronized mutation is valid or supported.

## Uncertainties

### Inferred intent

The structure suggests comparison by a two-word primary key, followed by a secondary unsigned word and address-based tie-breaking. Interpreting the primary pair as a logical unsigned 64-bit key is consistent with stable behavior.

The record domain, field meanings, intended callers, and reason for a positive self-comparison are not established.

### Documented-only unknowns

The following remain unresolved:

- Original record declarations and the meaning of unused bytes.
- Independently confirmed target ABI, word width, endianness, alignment, and pointer rules.
- Ownership of either record and responsibility for releasing its storage.
- Caller interpretation of the unsigned return values.
- Whether callers expect a conventional comparator.
- Whether mutable, volatile, device-backed, or concurrently accessed storage is possible.
- Exact machine-load count and ordering beyond the supplied source-level structure.
- Atomicity, synchronization, and thread-safety guarantees.
- Platform behavior for invalid or wrapped addresses.

The staged observations describe only what the supplied body supports. They do not establish a coherent snapshot or make otherwise-invalid memory access patterns valid.

## Evidence

Established static evidence from the supplied body for F-ef0c767f14fe includes:

- Addressed unsigned-word reads at offsets +0x0C, +0x10, and +0x14.
- Unsigned field comparisons and a final unsigned address comparison.
- Separate primary admission and reassessment stages.
- Repeated reads of primary and secondary fields.
- Conditional secondary reads and a short-circuit address tie-break.
- Only **0x00000001** and **0xFFFFFFFF** as normal return values.
- No explicit calls, memory writes, allocation, release, ownership transfer, or lifetime operation.

The stable-record ordering follows from those conditions under the stated validity and stability preconditions. Ownership and environmental guarantees do not follow from the address arguments or the comparison logic.

No runtime tests have been performed.
