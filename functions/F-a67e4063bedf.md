<!-- Trusted provenance: F-a67e4063bedf; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-a67e4063bedf

## Purpose

F-a67e4063bedf conditionally processes two target-associated values through construction-like, setup, and registration-like dependency calls. The first target comes from a lookup dependency. The second target is repeatedly read from the supplied context.

The established contract is the ordering of gates, memory observations, temporary-storage writes, dependency arguments, and returned-value flow. “Object,” “construction,” and “registration” below are descriptive labels, not guarantees about domain meaning, fresh creation, or ownership.

**Inferred intent:** the complementary placement of 320 and an external value, together with mode values 9 and 6, suggests coordinated behavior involving two targets. The source does not establish animation, movement, timing units, or any particular application domain.

## Interface

### Arguments and result

The function accepts three 32-bit arguments:

| Argument | Established use |
|---|---|
| Context address | Base for field reads and for forming embedded-structure addresses passed to dependencies. |
| Construction selector | Zero selects the direct path; any nonzero value selects the composed path. |
| Arrangement selector | Selects complementary temporary-record values and, on the second composed path, one context field used in arithmetic. |

Both selectors are tested only for zero. Negative values take the nonzero branches.

The function has no explicit result or status output. Normal return does not distinguish an initial gate rejection, a failed lookup, or completion of both processing sequences.

### Context and result layouts

Offsets are in bytes. Each listed field read is 32 bits.

| Location | Established use |
|---|---|
| Context +88 | Address passed to F-7e55b35f8edc; forming this address does not itself read an embedded field. |
| Context +116 | Address passed to F-5c81bf9e23ce on each composed path. |
| Context +132 | Read once per processing sequence, including on direct paths where its value is not used in construction. |
| Context +180 | Read for second-sequence arithmetic when the arrangement selector is nonzero. |
| Context +196 | Read for second-sequence arithmetic when the arrangement selector is zero. |
| Context +200 | Initial gate; subsequently reread for individual second-target arguments. |
| F-38aa1bb9efd0 result +232 | Read and supplied to the lookup dependency. |
| Each constructed-object result +4 | Read separately for three later consumers. |

Address-like values and temporary value slots are represented as 32-bit quantities. Each temporary record described below consists of four consecutive four-byte words.

### Preconditions and failure boundary

Every address actually dereferenced must support the corresponding read for execution to complete normally. Dependencies must accept the supplied values and temporary-storage addresses.

There is no visible context-null check, returned-address check before the +232 read, constructed-object check before a +4 read, or explicit arithmetic-error check. The source provides no usable failure convention for any dependency.

No property of F-8157a180d901 establishes that it allocates memory or that its integer arguments are byte counts.

## Behavior

### Entry gates and retained values

The first visible context access reads +200. If that value is zero, the function returns without calling a dependency or reading any of the three external configuration locations described below.

Otherwise, it:

1. Calls F-7a89f2f7fd9a and retains its result as the registration-context value.
2. Calls F-38aa1bb9efd0.
3. Reads the 32-bit field at +232 from that returned address.
4. Calls F-db754b1ab3c7 with the field value and zero.
5. Reads external configuration **A**, retaining this observation for possible use in the second temporary record.

The A read occurs even if the lookup result is zero. A zero lookup result then ends processing. No object-processing sequence begins.

A nonzero lookup result is retained as the first target. The first sequence runs before the second. There is no later gate that independently suppresses the second sequence.

### External configuration observations

A, B, and C denote three distinct external 32-bit storage locations. Their names do not imply units, signedness, immutability, or meaning.

| Configuration | Read behavior |
|---|---|
| A | Read after the lookup call, before its result is tested. If the lookup succeeds, read again while initializing the first record, regardless of the arrangement selector. |
| B | Read after the first F-a19ace379809 call and retained. On the first direct path only, read again for argument 4 of F-2698159acdb1. |
| C | Read separately as an argument to each F-aa408aa1a280 call. |

The **retained B observation** supplies the first composed path and the second construction path, whether direct or composed. The first direct call instead receives the **later B observation**.

Thus, successful direct-path execution contains two visible B reads. Successful composed-path execution contains one. These observations must not be treated as interchangeable merely because no explicit write to B appears in this function.

### Temporary records and their initialization

The first record has the following contents immediately before its F-a19ace379809 call:

| Word | Value |
|---|---|
| 0 | 0 |
| 1 | 0 |
| 2 | The second A observation if the arrangement selector is zero; otherwise 320 |
| 3 | 0 |

Initialization first writes the second A observation into word 2. A nonzero arrangement selector replaces that word with 320. The remaining visible initialization writes occur in the order word 3, word 0, then word 1, each with zero.

Consequently, the second A read still occurs when its value is subsequently overwritten by 320.

F-a19ace379809 receives the first target and the address of word 2. Later, F-a8c06eac92c0 receives the address of word 0 of the same record. The former is an interior address eight bytes beyond the latter.

The second record is prepared only after the entire first sequence has completed:

| Word | Value |
|---|---|
| 0 | 320 if the arrangement selector is zero; otherwise the earlier retained A observation |
| 1 | 0 |
| 2 | 0 |
| 3 | 0 |

Its visible writes initialize word 2, word 3, and word 1 to zero, in that order, followed by word 0 with the selected value.

F-a19ace379809 receives a fresh +200 read and the address of this record’s word 2. F-a8c06eac92c0 later receives its word-0 address.

The records’ initialized contents are not guaranteed to survive until setup: a dependency receives a writable interior address first. Neither its access extent nor the records’ interpretation is established.

### First construction path

After the first F-a19ace379809 returns, the function reads and retains B, then reads context +132.

On the **direct path**, F-2698159acdb1 receives six ordered arguments:

1. Context address.
2. Zero.
3. Retained first target.
4. A fresh read of B, not the retained B observation.
5. Zero.
6. One.

Its result becomes the first object value. The preceding +132 read still occurs, although its value is unused by this path.

On the **composed path**:

- F-8157a180d901 is called with 52.
- F-dff2a180229b receives that result, the previously read +132 value plus 500, retained B, and three zeros.
- The F-dff2a180229b result is stored in a temporary 32-bit slot.
- F-8157a180d901 is called with 84.
- F-104bb9725612 receives that result, zero, the first target, and the current intermediate-slot value.
- Its result becomes the first object value.
- F-5c81bf9e23ce receives context +116 and the address of the intermediate slot.

The last call occurs before first-object setup. It is unknown whether it copies, replaces, retains, or releases anything. The results from the calls with 52 and 84 are merely inputs to subsequent dependencies; fresh storage is not established.

### First setup and registration sequence

The following operations occur in order:

1. F-a8c06eac92c0 receives the context, first object, first target, first-record word-0 address, and zero.
2. F-8157a180d901 is called with 36.
3. F-3a954974c1bb receives that returned value, a read of first-object +4, zero, the first target, and mode **9**.
4. F-4b539170a43a receives the retained registration-context value and the first object.
5. F-aa408aa1a280 receives the registration-context value, another read of first-object +4, and a read of C.
6. First-object +4 is read again into a temporary 32-bit slot.
7. F-7e55b35f8edc receives context +88 and the address of that slot.

The result of F-3a954974c1bb is unused.

The three +4 reads are separate observations, not a single cached value. Intervening dependencies may change the field. The value supplied through the final temporary slot therefore need not equal either earlier observation.

### Second construction path

After second-record initialization, F-a19ace379809 is called with a fresh context +200 read and the record’s word-2 address. Context +132 is then read again.

On the **direct path**, F-2698159acdb1 receives:

1. Context address.
2. Zero.
3. Another fresh context +200 read.
4. Retained B from the first sequence.
5. Zero.
6. One.

Its result becomes the second object value. There is no additional B read here. The second +132 observation is unused on this path.

On the **composed path**, the arrangement selector chooses one context field:

- Zero selects +196.
- Nonzero selects +180.

The function subtracts that field value from 320 and multiplies the difference by 100. F-870c4fcf7295 receives that result and 320.

The returned value is multiplied by retained B. F-d7be9cfb4000 receives that product and 100.

The remaining sequence is:

- Call F-8157a180d901 with 52.
- Call F-dff2a180229b with that returned value, the arithmetic-helper result plus the second +132 observation plus 500, retained B, and three zeros.
- Store the result in the intermediate temporary slot.
- Call F-8157a180d901 with 84.
- Call F-104bb9725612 with that returned value, zero, a fresh +200 read, and the intermediate-slot value.
- Use its result as the second object value.
- Call F-5c81bf9e23ce with context +116 and the intermediate-slot address.

The helper argument shapes suggest scaling, but neither division nor percentage semantics are established.

### Second setup and registration sequence

The second object follows the first sequence’s dependency order, with these differences:

- F-a8c06eac92c0 receives the second record’s word-0 address and a fresh +200 target read.
- F-3a954974c1bb receives another fresh +200 target read and mode **6**.
- Object-field reads refer to the second object.
- F-aa408aa1a280 obtains a new observation of C.

The registration-context value remains the retained entry-time dependency result.

After the final F-7e55b35f8edc call, the function returns without an explicit status.

### Widths and evaluation order

Visible arithmetic uses 32-bit integer quantities: addition of 500, subtraction from 320, multiplication by 100, multiplication by retained B, and addition of the second derived offset.

No wider intermediate, range validation, saturation, or overflow handling is visible. Unbounded mathematical arithmetic is not an established model. Exact overflow behavior and the arithmetic dependencies’ signedness and rounding remain unresolved.

The source establishes the dependency-call sequence and the explicit reads and writes between calls. It does not establish the relative evaluation order of separate argument expressions within a single call. In particular, no order is promised between a +4 field read and the C read belonging to the same registration call.

## Side effects

Visible direct writes initialize or update temporary records and 32-bit slots. No direct write to context storage or either object’s fields appears.

Dependencies receive context addresses, target values, returned object values, embedded-structure addresses, and pointers into temporary storage. Any resulting mutation, retention, resource management, or external registration is dependency-defined.

The function visibly calls F-8157a180d901:

- With 36 once per completed object sequence.
- Additionally with 52 and 84 per object sequence on composed paths.

These are integer arguments, not established allocation sizes. Their returned values are forwarded as described above. Allocation, fresh creation, distinctness, ownership transfer, and eventual release are all unproven.

No explicit rollback, retry, or cleanup operation is identifiable from the supplied function. A later failure may follow earlier dependency effects; atomicity is not established.

The second target is not a snapshot of the initial +200 gate. Later reads can observe another value, including zero, without another gate.

Temporary addresses have invocation-local storage behind them. This function establishes no permission or lifetime guarantee for using those addresses after return.

## Dependencies

| Opaque ID | Established interaction |
|---|---|
| F-7a89f2f7fd9a | Returns the retained registration-context value. |
| F-38aa1bb9efd0 | Returns an address whose +232 field is read. |
| F-db754b1ab3c7 | Receives that field and zero; its result gates further processing and supplies the first target. |
| F-a19ace379809 | Receives a target and a temporary record’s word-2 address. |
| F-2698159acdb1 | Returns the object value on a direct path. |
| F-8157a180d901 | Receives 52, 84, or 36; its results feed other dependencies. |
| F-dff2a180229b | Returns the intermediate value on a composed path. |
| F-104bb9725612 | Returns the object value on a composed path. |
| F-5c81bf9e23ce | Receives context +116 and the intermediate-slot address. |
| F-a8c06eac92c0 | Receives context, object, target, record base, and zero. |
| F-3a954974c1bb | Receives a forwarded result, object +4, zero, target, and mode 9 or 6. |
| F-4b539170a43a | Receives registration context and object. |
| F-aa408aa1a280 | Receives registration context, object +4, and C. |
| F-7e55b35f8edc | Receives context +88 and a temporary field-value address. |
| F-870c4fcf7295 | Receives the scaled difference and 320. |
| F-d7be9cfb4000 | Receives the retained-B product and 100. |

No dependency is established to preserve global configuration, context fields, object fields, or ownership.

## Acceptance examples

These are conditional acceptance criteria, not executed tests.

1. **Initial gate rejection.** If the first +200 read is zero, there are no dependency calls and no A, B, or C reads.

2. **Lookup rejection.** If the initial gate passes but the lookup returns zero, the entry and lookup calls occur, followed by one A read. There are no B or C reads and no record-processing calls.

3. **Distinct B observations on the direct path.** Suppose retained B is observed as 7 and the later first-direct-call observation is 11. The first F-2698159acdb1 receives 11 as argument 4; the second receives 7. This criterion does not require or posit a particular cause for the differing observations.

4. **Successful read accounting.** With returning dependencies and valid accessed storage, the direct path has two A reads, two B reads, and two C reads. The composed path has two A reads, one B read, and two C reads. These count only reads visible in this function.

5. **Complementary records.** If the arrangement selector is zero, the first record initially contains zero, zero, the second A observation, zero; the second contains 320, zero, zero, zero. If nonzero, the first record’s word 2 is 320 and the second record’s word 0 uses the earlier A observation. The second A read occurs in both cases.

6. **Composed call arguments.** Under normally returning dependencies, successive F-8157a180d901 arguments are 52, 84, 36, 52, 84, 36. On the direct path they are 36, 36. No allocation or freshness property follows.

7. **Arithmetic boundary.** If the selected +180 or +196 field is 320, F-870c4fcf7295 receives zero and 320. Without helper contracts, neither subsequent helper result is prescribed.

8. **Mutable context and fields.** If dependencies change +132 between sequences, second composed construction uses the later observation. Changes to +200 affect later individual target reads without regating. Changes to an object’s +4 field between consumers affect the subsequent reads.

9. **Record mutation.** If F-a19ace379809 modifies accessible record storage through its supplied address, later setup receives that same record storage, not a reinitialized copy.

## Uncertainties

The following remain **documented_only**:

- Domain meanings, units, ranges, and validity constraints.
- Whether any dependency constructs a fresh object or allocates storage.
- Dependency failure conventions and behavior after invalid returned values.
- Arithmetic-helper semantics, signedness, rounding, and overflow behavior.
- Exact temporary-record interpretation and dependency access extent.
- Ownership, aliasing, identity, retention, and release of returned values.
- Whether embedded-structure operations copy, replace, retain, or release values.
- Mutations performed by dependencies.
- Thread safety, reentrancy, and synchronization requirements.
- Behavior after invalid addresses or arithmetic conditions outside supported ranges.

The absence of checks does not establish that invalid inputs are harmless or impossible.

## Evidence

This contract derives from static inspection of the supplied function representation. Established claims concern visible branches, 32-bit quantities, field offsets, constants, temporary layouts and writes, repeated configuration observations, dependency arguments, and call ordering.

The first direct path’s additional B read is distinct from the retained B observation. Calls with 52, 84, and 36 establish only argument values and returned-value flow, not allocation semantics.

Domain interpretations are explicitly identified as inferred intent. Missing dependency contracts remain unresolved.

No runtime tests have been performed.
