<!-- Trusted provenance: F-bfc162dd4e46; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-bfc162dd4e46

## Purpose

F-bfc162dd4e46 traverses a linked sequence of records, selects records whose three key fields equal three supplied arguments, and invokes F-88996a971ccf for each matching visit. Each invocation receives a fixed descriptor and the address of a temporary 32-bit value computed from the matching record and two shared numeric values.

The established behavior is conditional calculation and dependency invocation. The function does not return a match count, stop after the first match, or explicitly remove records.

Interpreting the calculation as a duration, elapsed time, or other physical quantity is **inferred intent**. The descriptor’s contents and the dependency’s externally visible behavior are not established.

## Interface

### Arguments and result

The function accepts three integer arguments represented as 32-bit values.

| Argument | Matching field |
|---|---|
| First | Record word at byte offset 24 |
| Second | Record word at byte offset 16 |
| Third | Record word at byte offset 28 |

Matching uses equality of the complete 32-bit values. No narrowing, masking, normalization, or range validation is shown. Comparison order differs from argument order: the function checks the first, third, then second argument.

The function has no return value. It provides no explicit success indicator, error code, count, or persistent result object.

### Shared inputs

The function uses:

- A shared location containing the initial record address.
- A shared reference-value location.
- A shared multiplier location.
- A fixed descriptor supplied to F-88996a971ccf.

Before reading the initial record address, the function obtains the address of the reference-value location and then the address of the multiplier location. These addresses are retained for traversal; the numeric contents are not cached at entry.

The numeric contents are read separately for each matching visit. Consequently, changes to those contents may affect later calculations. Changing a shared pointer slot from which an address was obtained does not itself replace the address already retained by this activation.

The initial record address is read once. Subsequent record addresses come from successor fields.

### Record layout

The accessed representation consists of four-byte words.

| Byte offset | Accessed role |
|---:|---|
| 0 | Successor address; zero terminates traversal |
| 8 | Numeric source value |
| 16 | Key compared with the second argument |
| 24 | Key compared with the first argument |
| 28 | Key compared with the third argument |

The successor is represented by a four-byte word interpreted as an address. No wider-pointer layout is established.

Offsets 4, 12, and 20 are not explicitly accessed. The complete record type, allocation size, alignment requirements, and ownership model are unknown. A fully evaluated candidate requires accessible fields through byte offset 31. Short-circuit evaluation skips some fields on a mismatch; this does not establish a supported truncated-record format.

### Preconditions for ordinary completion

The shared locations and every field actually accessed must remain valid for the relevant access. Any reached dependency invocation must return normally for traversal to continue.

A finite initial chain alone does not guarantee completion or a particular visit count. Dependency behavior may change links, matching fields, numeric inputs, or storage validity. No synchronization, record pinning, or protection against such changes is shown.

## Behavior

### Initialization and empty input

The function obtains the two shared numeric-location addresses, then reads the initial record address.

If that address is zero, traversal ends immediately. No record fields or shared numeric contents are read, and F-88996a971ccf is not invoked. Obtaining the shared numeric-location addresses still precedes this empty-sequence decision.

### Selection

For each nonzero current record address, selection proceeds in this order:

1. Read the key at offset 24 and compare it with the first argument.
2. If that comparison succeeds, read the key at offset 28 and compare it with the third argument.
3. If both preceding comparisons succeed, read the key at offset 16 and compare it with the second argument.

A failed comparison prevents the remaining key-field reads for that visit. It also prevents the numeric calculation and dependency invocation.

A mismatch does not terminate traversal. The function still reads the current record’s successor and advances.

A record matches only when all three equalities succeed. There is no deduplication by address or by key values.

### Per-match calculation

For each match, the function calculates:

**multiplier × (record source value − reference value)**

The source value comes from offset 8 of the current record. The reference and multiplier come from the shared locations whose addresses were obtained before traversal.

The result is written into a temporary 32-bit integer object. No widened accumulator, saturation, bounds check, or explicit arithmetic-error handling is shown.

Where the operands and operations have an unambiguous representable result under the applicable arithmetic interpretation, the temporary contains that mathematical result. The representation establishes four-byte operands and destination storage, but it does not establish a complete portable overflow contract. In particular, neither guaranteed modulo behavior nor a defined signed-overflow result should be inferred.

The relative evaluation order of the source-value, reference-value, and multiplier reads is not reliably established by the supplied expression. The contract therefore does not prescribe which of these three reads occurs first. No atomic snapshot across them is shown.

### Dependency invocation

After writing the calculated value, the function invokes F-88996a971ccf with:

- The fixed descriptor.
- The address of the temporary 32-bit value.

The numeric value is passed indirectly, not as a direct integer argument. No dependency result is consumed, and no subsequent use of a dependency-modified temporary value is shown.

There is one invocation per matching visit that reaches the call. This is not necessarily one invocation per distinct record: revisiting a matching record can invoke the dependency again.

### Advancement

After completing the current visit, including any dependency invocation, the function reads the current record’s successor at offset 0.

The successor is not saved before the dependency call. If the dependency returns after changing that field, the next step uses the successor available at the subsequent read.

A zero successor ends traversal. A nonzero successor becomes the next current record.

The root is not reread during advancement. Changing the root alone therefore does not restart or redirect this activation, provided the dependency returns and the retained current record and successor chain remain valid.

### Termination and abnormal conditions

There is no visible cycle detection, visit limit, timeout, malformed-link handling, or recovery path.

A stable valid cycle prevents return if each visit completes and no external action breaks the cycle. Invalid addresses, invalidated storage, nonreturning dependency calls, and arithmetic cases outside the established semantics have no defined recovery behavior in this contract.

## Side effects

### Direct effects

The function explicitly:

- Obtains the shared numeric-location addresses.
- Reads the initial record address.
- Reads key fields in the short-circuit order described above.
- Reads numeric operands for matching visits.
- Writes a calculated value to temporary local storage.
- Passes that storage’s address to F-88996a971ccf.
- Reads the current successor after processing each visit.

It does not explicitly write record fields, the root, or either shared numeric value. No allocation, release, unlinking, or ownership transfer is shown.

These statements describe this function’s direct operations. They do not establish that the invocation as a whole leaves shared state unchanged.

### Dependency-mediated changes

F-88996a971ccf has no supplied behavioral contract. It must not be assumed to preserve:

- The current record or other records.
- Successor links or later matching fields.
- Shared numeric contents.
- Shared pointer slots.
- The temporary value.
- Storage validity or ownership.
- Normal control flow.

If the dependency changes numeric contents at the retained addresses, later matching calculations can observe the changes.

If it changes a successor, traversal can skip records, reach additional records, end early, or enter a cycle. If it changes later keys, an initially matching record may cease to match, or an initially nonmatching record may become a match.

If it releases or invalidates the current record, the post-call successor read may access invalid storage. No protective copy or lifetime mechanism is shown.

### Temporary storage and ownership

The address passed to the dependency refers to temporary storage belonging to this activation. The function does not create a separately owned result object for each match.

Successive invocations need not receive distinct numeric storage. The value is written again for each matching calculation, so retaining its address does not establish a stable historical result.

No ownership transfer is established. Whether the dependency copies the value, modifies it, retains the address, or requires a particular lifetime is **documented_only**. The supplied evidence does not justify assuming the address remains valid after this activation ends.

## Dependencies

### F-88996a971ccf

**Established call-site interface:** receives a fixed descriptor and the address of a temporary 32-bit integer.

**Invocation condition:** all three key comparisons succeed and the numeric calculation reaches completion.

**Ordering:** the invocation follows the temporary write and precedes the current record’s successor read.

**Result handling:** no return value is used by F-bfc162dd4e46.

**Unknown behavior:** descriptor interpretation, formatting, output destination, error signaling, global mutation, record mutation, memory allocation or release, pointer retention, ownership requirements, and normal-return guarantees.

The descriptor’s exact contents are not established as independently readable data. No exact output text, punctuation, units, or line endings can be required.

No other called dependency is visible in the supplied function.

## Acceptance examples

These are conditional expectations derived from static evidence, not executed tests. Each example requires valid storage for every access it reaches and arithmetic within the established interpretation.

### Empty sequence

Given a zero initial record address and valid access to the shared locations needed during initialization:

- No record field is read.
- No numeric calculation occurs.
- F-88996a971ccf is not invoked.
- The function returns.

### First-key mismatch

Given one valid record whose offset-24 key differs from the first argument and whose successor is zero:

- Offset 24 is read for selection.
- Offsets 28 and 16 are not read for selection.
- Offset 8 and the shared numeric contents are not read for calculation.
- The successor is read.
- The function returns without a dependency invocation.

### Third-argument mismatch

Given a valid record whose offset-24 key matches the first argument but whose offset-28 key differs from the third argument:

- Offset 24 is read before offset 28.
- Offset 16 is not read for selection.
- No calculation or dependency invocation occurs for this visit.
- Traversal advances through the successor.

### Second-argument mismatch

Given a valid record whose offset-24 and offset-28 keys match, but whose offset-16 key differs from the second argument:

- All three selection fields are read in order: 24, 28, 16.
- No numeric calculation or dependency invocation occurs.
- The successor is read next.

### Complete match

Given matching keys and numeric reads yielding source value 120, reference value 100, and multiplier 3:

- The temporary contains 60 when the dependency is invoked.
- The dependency receives the fixed descriptor and the temporary’s address.
- If the dependency returns normally, the successor is read afterward.

This does not assert that the dependency preserves the value or produces any particular external output.

### Multiple matches with an intervening mismatch

Suppose three initially linked records have matching, nonmatching, and matching keys respectively, followed by a zero successor.

Exactly two dependency invocations follow only under these additional conditions:

- Every reached dependency invocation returns normally.
- All subsequently accessed records and shared locations remain valid.
- Successor reads continue to lead through those three records and then zero.
- Selection reads continue to yield the stated matching conditions.
- The two matching calculations complete under the applicable arithmetic semantics.

Under those conditions, the invocations occur in first-record then third-record order. The intervening mismatch does not terminate traversal.

Initial acyclicity and initial keys alone are insufficient: the first dependency invocation could change the successor, invalidate storage, alter later keys, or fail to return.

### Updated numeric state

Suppose the first matching calculation reads source value 120, multiplier 3, and reference value 100. Its temporary is 60.

Suppose its dependency invocation changes the reference value to 110 and returns. A later temporary is 30 only if a second matching calculation is actually reached and reads:

- Source value 120.
- Multiplier 3.
- Reference value 110.

The later access must remain valid, and the calculation must complete. Merely keeping the reference at 110 is insufficient: the dependency could also change traversal, later keys, the source value, or the multiplier.

### Updated successor

Given a matching current record, if the dependency returns normally, leaves the current record valid, and changes its successor to zero, traversal ends at the post-call successor read.

If instead that read yields another valid record address, that address becomes the next candidate. Subsequent matching behavior depends on the fields actually read there.

### Stable cycle

Given a valid cyclic successor chain that remains cyclic, with all reached operations completing normally:

- The function does not return.
- Every matching visit that completes its calculation invokes the dependency.
- Repeated visits to the same matching address are not suppressed.

## Uncertainties

The following remain **documented_only**:

- Exact arithmetic signedness, conversions, and overflow behavior for all operands.
- Descriptor contents and resulting external output.
- Alignment, address validity, and synchronization guarantees.
- Effects of concurrent mutation or data races.
- Dependency ownership, pointer retention, and lifetime requirements.
- Dependency errors, nonlocal transfers, and return guarantees.
- Whether the surrounding system guarantees finite, acyclic traversal.
- The domain meaning and units of the numeric calculation.

The function shows no explicit validation of shared addresses or record links and no arithmetic-error recovery. Absence of validation does not define a portable result for invalid conditions.

## Evidence

**Established static evidence:** a four-byte-word record layout; short-circuited key reads at offsets 24, 28, and 16; per-match arithmetic using offset 8 and two shared numeric contents; a dependency invocation receiving a fixed descriptor and a temporary address; and successor traversal through offset 0 after the invocation.

**Inferred intent:** selecting records for presentation of a derived numeric quantity. Timing semantics, units, formatting, and output destination remain unestablished.

**Verification status:** no runtime tests have been performed. This contract is based solely on the supplied static representation. Dependency internals, descriptor contents, and execution-environment guarantees were not available.
