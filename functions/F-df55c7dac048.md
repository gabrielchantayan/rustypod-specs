<!-- Trusted provenance: F-df55c7dac048; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-df55c7dac048

## Purpose

F-df55c7dac048 validates a requested sequence of 32-bit values against an indirectly stored count and an inclusive numeric range. If validation and any required preparation succeed, it selects an allocation or existing-storage comparison path, records the requested count, invokes an unidentified external dependency, and writes a context byte to 1. Subsequent calls depend on whether storage was newly allocated or existing contents differed.

The external invocation supplies the current destination-address field, the source address, and the requested count shifted left by two in 32-bit arithmetic. **The supplied source does not establish that this dependency copies bytes or modifies destination storage.** Its copy-like purpose is inferred from its arguments, not established behavior.

The apparent broader intent is to maintain a numeric sequence and coordinate related state. This interpretation does not establish the meaning, ownership, or lifetime of the referenced objects.

## Interface

### Inputs

The function has three inputs:

| Input | Width | Established use |
|---|---:|---|
| Context address | 32 bits | Base address for context-field reads and writes; passed to several dependencies. |
| Requested count | 32 bits, unsigned | Expected-count comparison, loop bound, allocation argument, recorded count, and external-call length calculation. |
| Source address | 32 bits | Base address for four-byte element reads; passed unchanged to the unidentified external dependency. |

Source values are read as 32-bit quantities. Validation accepts exactly the bit patterns representing signed integers from −65,536 through +65,536, inclusive. Existing-storage comparisons test equality of complete 32-bit values.

There are no explicit checks for null input addresses, alignment, accessible object sizes, destination capacity, or address arithmetic overflow.

### Context layout

Offsets are byte offsets from the context address. Only accesses established for this function are described.

| Offset | Width | Use |
|---|---:|---|
| `0x64` | 32 bits | Read near entry and retained as an argument for later dependencies. |
| `0x68` | 32 bits | Read for a post-invocation dependency on the newly allocated path. |
| `0x284` | 32 bits | Tested after the external invocation and flag write; explicitly cleared on one branch. |
| `0x29C` | 8 bits | Written to 0 near entry and to 1 after the external invocation returns. |
| `0x2A0` | 32-bit address | Address of the associated state block. |

### Associated state-block layout

| Offset | Width | Use |
|---|---:|---|
| `0x00` | 32 bits | Receives the requested count on the continuing path. |
| `0x04` | 32-bit address | Zero selects allocation; otherwise identifies storage used for comparison. Read as the first argument to the external invocation. |
| `0x08` | 32-bit address | Points to the 32-bit expected count. |
| `0x24` | 32 bits | Zero selects a preparation call; nonzero skips that call. |

These layouts do not establish object types or non-aliasing relationships.

### Return value

The return value consists of two 32-bit words assembled into a 64-bit result:

- **Low word:** the final 32-bit status representation.
- **High word:** the requested count, except that reaching the allocation call changes it to zero.

The low word initially contains zero. The function explicitly assigns status 6 for either an expected-count mismatch or a rejected source value. Nonzero statuses from initialization, preparation, or allocation terminate processing and are returned in the low word.

The high word remains zero after an allocation attempt, whether allocation reports success or failure. It is not restored to the requested count.

### Preconditions for ordinary completion

The context and every location actually accessed must support the indicated reads and writes. A successful initialization must leave a usable state-block address in the context. The retained block must remain valid across dependencies that precede its subsequent use.

For nonzero counts, source storage must support the validation reads. An existing destination must support comparison reads up to the first mismatch or the end of the sequence. Requirements imposed by the unidentified external dependency are unknown.

These are conditions needed for defined, ordinary execution, not validations performed by this function. In particular, a zero count eliminates element reads in the two loops but does not eliminate dependencies or establish that a null source is acceptable to them.

## Behavior

### Entry and initialization

The function initializes its status to zero, reads the 32-bit context field at `0x64`, and then writes zero to the byte at `0x29C`.

The value captured from `0x64` is retained. Subsequent changes to that context field do not refresh the argument used by later allocation and invalidation-related calls.

The function tests the address at context offset `0x2A0`:

- If nonzero, initialization is skipped.
- If zero, F-609bb7a48db0 is called with the context address and a zero second argument.
- A nonzero returned status terminates processing.
- A zero returned status permits continuation.

On continuation, the function loads the state-block address from `0x2A0`. This load occurs after the initial test and any initialization call. There is no separate null check following successful initialization.

An initialization error leaves the high return word equal to the requested count.

### Expected-count check

Using the loaded state block, the function reads the address at block offset `0x08` and then reads the 32-bit expected count through that address.

If the expected count differs from the requested count, the function assigns status 6 and stops. No source-element validation, preparation, destination comparison, allocation, or unidentified external invocation follows on this path.

The function does not revalidate the expected count later, including after dependencies that might modify it.

### Source validation

If the counts match, the function reads source elements in increasing index order, beginning at index zero.

For each four-byte element, it adds `0x10000` using unsigned 32-bit arithmetic. The value is rejected if the resulting unsigned quantity exceeds `0x20000`.

This accepts exactly:

**−65,536 ≤ signed interpretation of the element ≤ +65,536**

The endpoints are included. The unsigned addition wraps modulo \(2^{32}\); this wrap is what allows the accepted negative representations to pass.

The first rejected element sets status 6 and stops processing. No later element is read by the validation loop. For a zero requested count, this loop performs no source-element reads.

Validation is an observation of the values at that time. It does not establish that later comparison reads or the external dependency will observe the same contents.

### Preparation

After successful validation, the function reads block offset `0x24`.

- Nonzero: skip preparation.
- Zero: call F-03409a4f0817 with the context.
- Nonzero returned status: terminate processing.
- Zero returned status: continue.

The function retains the state-block address loaded before this call. It does not reload the address from context offset `0x2A0` afterward.

Consequently, later accesses use the retained block even if preparation changes the context’s block-address field. This access pattern does not establish that preparation preserves the retained block’s lifetime or contents.

### Destination selection and classification

The function next reads block offset `0x04`.

**Zero destination address:**

1. Set the high return word to zero.
2. Call F-6dd003795db7 with, in order:
   - the entry-captured value from context offset `0x64`;
   - 4;
   - 0;
   - the requested count;
   - 0;
   - an address through which the dependency can update the local 32-bit status.
3. Store the dependency’s 32-bit result into block offset `0x04`.
4. Inspect the status.
5. If nonzero, terminate processing; otherwise classify the path as newly allocated storage.

The result is stored even when the status indicates failure. The function does not independently reject a zero result when the status is zero. The allocation-like purpose of this dependency does not establish ownership or capacity guarantees.

**Nonzero destination address:**

The function compares source and destination elements at increasing indices. At each visited index, it tests equality of the two 32-bit values. It stops at the first unequal pair or after the requested count.

- No unequal pair: classify as unchanged existing storage.
- Unequal pair encountered: classify as changed existing storage.

Both storage locations are read for each comparison that occurs; no additional ordering guarantee between the two operand reads is established here.

These comparisons read source values again after validation. For zero count, no comparison element is read and the classification is unchanged.

### Count write and unidentified external invocation

After successful allocation or existing-storage comparison, the function:

1. Writes the requested count to block offset `0x00`.
2. Calls D-01 with:
   - the **current** value at block offset `0x04`;
   - the original source-address argument;
   - the requested count shifted left by two in 32 bits.
3. After D-01 returns normally, writes 1 to context byte `0x29C`.

D-01 is an opaque document-local identifier for the external dependency whose identity was not supplied.

The first argument is read from the block for this invocation; the contract does not replace it with an assumed preserved allocation result or comparison address. The count-field write precedes this read.

The invocation occurs even when existing contents compare equal and even when the requested count is zero. No return value from D-01 is used to determine status or whether to write the context byte.

The third argument is a 32-bit quantity, not a checked or widened multiplication:

\[
L = (4 \times \text{requested count}) \bmod 2^{32}.
\]

For example, a count of `0x40000000` produces a third argument of zero, if execution reaches this point. The validation and comparison loops remain controlled by the original count. Their addressing also lacks explicit overflow protection.

**No destination-data write is established by this invocation alone.** Whether D-01 copies bytes, modifies destination storage, reads either supplied address, or performs another operation is unknown.

### Post-invocation handling

After writing 1 to `0x29C`, the function reads context offset `0x284`.

If that field is zero, no further explicit action occurs. If it is nonzero, classification determines the remaining behavior:

| Classification | Behavior |
|---|---|
| Newly allocated | Read the current value at context offset `0x68`; call F-73b66ba1e5b1 with the context and that value. |
| Changed existing storage | Call F-bf7242c98d06 with the entry-captured value from `0x64`; after normal return, write zero to context offset `0x284`. |
| Unchanged existing storage | Make neither of these calls and do not explicitly clear `0x284`. |

Neither call contributes an explicit status used in the returned result. Their other effects and any non-returning behavior are unknown.

## Side effects

The function’s explicit writes occur in this conditional order:

1. Clear context byte `0x29C`.
2. If allocation is attempted, store its result into block offset `0x04`, before checking allocation status.
3. On the continuing path, write the requested count to block offset `0x00`.
4. After D-01 returns, set context byte `0x29C` to 1.
5. On the changed-existing-storage branch with a nonzero `0x284`, clear that field after F-bf7242c98d06 returns.

D-01 is invoked between the count write and the flag write. Its invocation is observable, but a copy or destination-content modification is **not** an established side effect.

Errors before allocation cause no explicit destination-address replacement, count-field update, or D-01 invocation. An allocation error can leave a replaced destination-address field while skipping the count write and D-01.

There is no explicit rollback. Dependencies may have effects before returning an error.

The function contains no explicit source release or destination deallocation. This does not establish that dependencies preserve or release either resource.

The flag writes are ordered events, not unconditional final-value guarantees. Dependencies can access the context directly or indirectly, and their preservation of the flag is unknown. Likewise, aliasing among supplied objects can affect the consequences of field writes; disjoint storage is not checked.

## Dependencies

| Opaque ID | Established invocation and result use |
|---|---|
| F-609bb7a48db0 | Called with context and zero when the initial block address is zero. Its nonzero status stops processing. |
| F-03409a4f0817 | Called with context after validation when block offset `0x24` is zero. Its nonzero status stops processing. |
| F-6dd003795db7 | Receives the captured context value, 4, zero, requested count, zero, and a status-output address. Its result is stored before status inspection. |
| D-01 | Unidentified external dependency; this label is document-local. Receives the current block field at `0x04`, the source address, and the wrapped shifted count. Its result is not used. Copying and destination modification are unknown. |
| F-73b66ba1e5b1 | Called after D-01 and the flag write on the newly allocated path when `0x284` is nonzero. Receives context and the current field at `0x68`. |
| F-bf7242c98d06 | Called after D-01 and the flag write on the changed-existing-storage path when `0x284` is nonzero. Receives the entry-captured field at `0x64`; an explicit field clear follows normal return. |

No dependency is assumed to preserve globals, memory contents, object lifetimes, or ownership.

## Acceptance examples

These are conditional criteria derived from static evidence, not executed tests. Unless stated otherwise, assume accessible, suitably separated storage, normal dependency returns, successful required preparation, and no dependency mutation that changes the stated observations.

1. **Count mismatch.** Expected count 3 and requested count 2 produce low word 6 and high word 2. The entry flag clear occurs. No source validation or D-01 invocation follows.

2. **Inclusive range.** With matching count 3, the values −65,536, 0, and +65,536 all pass validation. This establishes permission to continue, not successful allocation or destination modification.

3. **Rejected value.** Either −65,537 or +65,537 produces status 6 when encountered during validation. Later validation elements are not read. Destination comparison and D-01 are not reached.

4. **Equal existing contents.** With count 2 and equal valid sequences, the function records count 2, invokes D-01 with the current destination field, source address, and third argument 8, then writes the flag to 1. Unchanged classification selects neither post-invocation dependency. The return words are high 2 and low 0. No assertion about copied bytes follows.

5. **Different existing contents.** With count 2 and a differing valid sequence, D-01 receives third argument 8. If `0x284` is nonzero after the flag write, F-bf7242c98d06 receives the entry-captured value and is followed by an explicit clear of `0x284`.

6. **Successful allocation.** With count 2 and an initially zero destination field, the allocation result is stored. Zero allocation status permits the count write and D-01 invocation with third argument 8. The return high word is zero. If `0x284` is nonzero after the flag write, F-73b66ba1e5b1 is called. This does not establish that allocated storage receives source contents.

7. **Allocation error.** A nonzero allocation status is returned in the low word; the high word is zero. The allocation result has already been stored, but no count write or D-01 invocation follows.

8. **Zero count.** Matching zero counts cause no validation reads. With existing storage, comparison performs no element reads and classifies unchanged; count zero is recorded and D-01 is still invoked with third argument zero. With missing storage, allocation is still attempted. D-01’s zero-length address requirements remain unknown.

9. **D-01 with no destination modification.** If D-01 returns normally without changing destination storage, the function still writes the flag to 1 and proceeds to classification-dependent handling. Such behavior does not contradict the supplied evidence; the caller neither checks nor establishes a copy.

## Uncertainties

The following remain **documented_only**:

- D-01’s identity and semantics, including whether it copies bytes or modifies destination storage at all.
- D-01’s address validity, overlap, zero-length, failure, and partial-effect rules.
- Concrete object types and domain meanings of the fields.
- Allocation ownership, capacity, failure-result conventions, and zero-count behavior.
- Dependency effects on globals, flags, source contents, retained blocks, or object lifetimes.
- Aliasing restrictions imposed by an external contract.
- Concurrency behavior; no synchronization is visible between validation, comparison, and invocation.
- External limits that might prevent arithmetic wrap or excessive element reads.
- Exceptions, traps, nonlocal exits, and other failures outside the explicit status mechanism.

No runtime tests have been performed.

## Evidence

Established static evidence is limited to the supplied function representation: field offsets and widths, conditional calls, unsigned range validation, ordered explicit writes, allocation-result storage before status inspection, equality comparison, the wrapped third argument to D-01, and two-word return construction.

The external invocation establishes its arguments and position in the control flow. It does not establish a byte transfer or destination-content mutation.

Descriptions involving sequence maintenance, allocation, preparation, or invalidation are inferred intent where they exceed those observed operations. Dependency implementations and external memory contracts were not supplied; their unknown behavior is not used to strengthen guarantees.
