<!-- Trusted provenance: F-ef1a85b90b2c; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-ef1a85b90b2c

## Purpose

F-ef1a85b90b2c traverses a collection associated with an input object, conditionally accumulates four-word geometry from its members, and compares the resulting horizontal and vertical spans with spans stored in that object. A qualifying mismatch invokes the corresponding axis-specific dependency.

The established behavior is filtering, accumulation through opaque dependencies, conditional normalization, and ordered mismatch handling. Interpreting the accumulated values as a bounding rectangle, or interpreting the axis-specific calls as layout invalidation, is inferred intent rather than a demonstrated dependency contract.

The function has no explicit return value. Observable effects arise from dependency calls and the memory those dependencies access or modify.

## Interface

### Input

The function accepts one address-bearing argument identifying a parent object. The supplied representation uses a 32-bit integer for this argument and for address calculations. A portable pointer-sized interface is not established.

The parent must provide readable storage for the fields below and a collection representation beginning at byte offset `0xA8`.

| Parent offset | Width | Established use |
|---|---:|---|
| `0x50` | 32 bits | First horizontal eligibility value |
| `0x54` | 32 bits | Flags paired with `0x50` |
| `0x5C` | 32 bits | First vertical eligibility value |
| `0x60` | 32 bits | Flags paired with `0x5C` |
| `0x68` | 32 bits | Second horizontal eligibility value, evaluated first |
| `0x6C` | 32 bits | Flags paired with `0x68` |
| `0x74` | 32 bits | Second vertical eligibility value, evaluated first |
| `0x78` | 32 bits | Flags paired with `0x74` |
| `0x80` | 32 bits | Stored horizontal start |
| `0x84` | 32 bits | Stored vertical start |
| `0x88` | 32 bits | Stored horizontal end |
| `0x8C` | 32 bits | Stored vertical end |
| `0xA8` | Unknown | Collection passed to the iteration initializer |

Each yielded collection member must identify readable storage containing a 32-bit flags word at member offset `0x48`. Additional member storage requirements are determined by F-450d2038c61f.

There is no explicit null check, range check, object-type check, or pointer-validity check. Valid parent, collection, and yielded-member representations are preconditions.

### Temporary layouts

The function supplies dependencies with three local storage areas:

- An eight-byte iteration-state area, not explicitly initialized before its initialization dependency.
- A 16-byte candidate area, not explicitly initialized before geometry extraction. This area is reused across members.
- A 16-byte accumulator, initialized as four zero-valued 32-bit words.

For describing the accumulator, this contract names its four consecutive words:

1. Horizontal start, at byte offset `0`.
2. Vertical start, at byte offset `4`.
3. Horizontal end, at byte offset `8`.
4. Vertical end, at byte offset `12`.

These names describe how the function uses the words after accumulation. They do not establish the candidate format or the behavior of the accumulation dependency.

### Output and arithmetic

There is no explicit result value or status channel.

The horizontal accumulated span is the third accumulator word minus the first. The vertical accumulated span is the fourth minus the second. Stored parent spans are computed from the corresponding end and start fields.

The supplied representation uses signed 32-bit operands for these subtractions. Overflow behavior is not independently established. Acceptance examples below use differences representable as signed 32-bit values; they do not require a particular overflow interpretation.

## Behavior

### 1. Initialize accumulation and iteration

All four accumulator words are set to zero before any dependency is called.

F-8ca21d3a8cca is then invoked with:

- The eight-byte local iteration-state area.
- The address of the parent collection at offset `0xA8`.

No initialization result is inspected.

### 2. Traverse and filter members

F-b38a3aba0e18 is repeatedly called with the iteration-state area and an output location for a member address. A zero result ends traversal. Every nonzero result enters the member-processing path; it need not equal one.

For each yielded member:

1. Read its 32-bit flags at offset `0x48`.
2. If both bits selected by mask `0x600` are set, skip all further processing of that member.
3. Otherwise invoke F-450d2038c61f with the member address and the 16-byte candidate area.
4. Invoke F-ddca41e891a8 with the candidate area.
5. If that result is nonzero, skip accumulation of this candidate.
6. If that result is zero, read the member flags at offset `0x48` again.
7. If both bits selected by mask `0x1800` are set in this second read, skip accumulation.
8. Otherwise invoke F-12cb3c413baa with the accumulator and candidate areas.

Both flag conditions require the entire respective mask to be present. One selected bit alone is insufficient to exclude a member.

The two member-flag reads are separate observations. Calls made between them may change the member or other state. Consequently, the second test must not be treated as a test against a cached copy of the first flags value.

The candidate predicate precedes the second flags test. A nonzero candidate-predicate result suppresses the second flags read entirely.

There is no direct candidate reset between iterations. Correct candidate production therefore depends on F-450d2038c61f’s actual contract; complete initialization of all candidate bytes is not demonstrated here.

### 3. Finalize traversal

After the first zero result from F-b38a3aba0e18, invoke F-d0d3f100effc with the iteration-state area.

This finalization occurs even if traversal yields no members. It occurs before the final accumulator predicate and before either axis comparison.

The supplied behavior contains no explicit exceptional-unwind path. If a dependency does not return normally, later finalization or notifications are not established.

### 4. Normalize the accumulator conditionally

Invoke F-ddca41e891a8 with the accumulator.

- A zero result leaves the accumulator as it exists after that call.
- A nonzero result causes all four accumulator words to be written as zero.

The dependency receives the accumulator’s address. Its possible mutation of that storage is not excluded. Thus, on a zero result, the subsequent comparisons use the post-call words, not necessarily their values immediately before the call.

The same dependency is used earlier for candidate filtering. Its semantic meaning—such as invalidity or emptiness—is unknown.

### 5. Compare and conditionally notify the horizontal axis

Compare:

- Parent word at `0x88` minus parent word at `0x80`.
- Accumulator horizontal end minus accumulator horizontal start.

If these spans are equal, perform no horizontal eligibility checks and make no horizontal notification call.

If they differ, evaluate these alternatives in order:

1. Read the parent value at `0x68`. Only if it is zero, read flags at `0x6C` and test bit `0x20`.
2. If the first alternative does not qualify, read the parent value at `0x50`. Only if it is zero, read flags at `0x54` and test bit `0x20`.

An alternative qualifies when its value is zero and its paired flags contain bit `0x20`.

If either alternative qualifies, invoke F-065fd89ae7c4 with the parent address. If the first alternative qualifies, do not evaluate the second alternative.

The source establishes these short-circuit conditions, but not a total temporal order among every operand read inside the span comparison itself.

### 6. Compare and conditionally notify the vertical axis

After the horizontal path completes, compare:

- Parent word at `0x8C` minus parent word at `0x84`.
- Accumulator vertical end minus accumulator vertical start.

If these spans are equal, perform no vertical eligibility checks and make no vertical notification call.

If they differ, evaluate these alternatives in order:

1. Parent value at `0x74` equals zero and paired flags at `0x78` contain bit `0x20`.
2. Otherwise, parent value at `0x5C` equals zero and paired flags at `0x60` contain bit `0x20`.

The same short-circuit rules apply: paired flags are read only when the preceding value is zero, and the second alternative is skipped when the first qualifies.

If either alternative qualifies, invoke F-d25031d00f00 with the parent address.

The vertical decision occurs after any horizontal notification. It is not based on an entry-time snapshot of the parent. Changes made by F-065fd89ae7c4 may therefore affect the later vertical comparison and eligibility checks.

## Side effects

### Direct writes

The function directly writes only temporary storage:

- Initial zero values into the accumulator.
- A possible second set of four zero writes after the final accumulator predicate.

Member-address output, iteration-state changes, candidate production, and accumulation are performed through dependencies.

No direct parent-field or member-field write is shown.

### Indirect effects and ownership

Dependencies receive addresses of the parent, members, collection storage, and local temporary areas. Their effects are not restricted by this function’s visible behavior.

In particular, this contract does not assume that dependencies:

- Preserve global state.
- Preserve parent or member fields.
- Leave input buffers unchanged.
- Allocate or release no memory.
- Preserve collection membership during traversal.
- Retain no pointers.

The function itself contains no explicit allocation or deallocation. Passing an address does not establish ownership transfer. The local buffers provide no demonstrated lifetime guarantee beyond the invocation.

F-d0d3f100effc is an explicit iteration-finalization call, but whether it releases resources, and which resources it owns, remain unknown. No separate candidate or accumulator finalizer is called.

### Errors and completion

There is no explicit error return, retry, diagnostic, or rollback.

The iterator’s zero result terminates traversal without distinguishing exhaustion from failure. Predicate results control filtering and normalization, not an exposed error channel.

At most one call to each axis-specific dependency occurs during a normally completing invocation. If both are called, the horizontal dependency precedes the vertical dependency.

## Dependencies

| Opaque ID | Established role at this call site |
|---|---|
| F-8ca21d3a8cca | Receives local iteration storage and the parent collection address before traversal. |
| F-b38a3aba0e18 | Receives iteration storage and member-output storage; zero ends traversal, nonzero enables member processing. |
| F-d0d3f100effc | Receives iteration storage once after normal traversal termination. |
| F-450d2038c61f | Receives a member and candidate storage after the first flags gate passes. |
| F-ddca41e891a8 | Tests candidate storage and later accumulator storage; zero and nonzero results have distinct caller-defined consequences. |
| F-12cb3c413baa | Receives accumulator and candidate storage after all member gates pass. |
| F-065fd89ae7c4 | Receives the parent when the horizontal mismatch and eligibility conditions hold. |
| F-d25031d00f00 | Receives the parent when the vertical mismatch and eligibility conditions hold. |

“Iteration,” “candidate,” and “accumulator” describe call-site roles. The dependencies’ internal algorithms, additional state access, failure modes, and ownership rules are not established.

## Acceptance examples

These are conditional static examples, not runtime test results. Unless stated otherwise, assume dependencies return normally and do not alter unrelated decision inputs.

1. **No yielded members.**  
   If the iterator immediately returns zero and the final accumulator predicate leaves the initially zero accumulator unchanged, both accumulated spans are zero. Each axis notifies only if its stored span is nonzero and its own eligibility condition qualifies.

2. **First mask excludes a member.**  
   With member flags `0x600` at the first flags read, neither candidate extraction nor candidate testing nor accumulation occurs for that member. Unrelated additional flag bits do not change this exclusion.

3. **A partial first mask permits extraction.**  
   With first-read flags `0x200` or `0x400`, the first gate passes. Later filtering still determines whether accumulation occurs.

4. **Candidate predicate rejects further processing.**  
   If extraction occurs and F-ddca41e891a8 returns any nonzero value for the candidate, there is no second member-flags read and no accumulation call for that candidate.

5. **Second mask excludes accumulation.**  
   If the candidate predicate returns zero and the second flags read contains all of `0x1800`, accumulation is skipped. A second-read value of only `0x800` or only `0x1000` does not satisfy that exclusion.

6. **Flags change between gates.**  
   If the first read is zero, candidate testing returns zero, and the second read is `0x1800`, extraction and testing occur but accumulation does not.

7. **Final normalization overrides accumulated words.**  
   If the final accumulator predicate returns nonzero, all four words become zero before span comparisons, regardless of their prior values.

8. **Equal spans suppress eligibility reads.**  
   If the stored horizontal span and accumulated horizontal span both equal 40, no horizontal eligibility fields are consulted and F-065fd89ae7c4 is not called.

9. **Only span equality matters.**  
   Stored horizontal endpoints of 10 and 50 match accumulated endpoints of 100 and 140 for the horizontal decision. Different positions alone do not cause notification.

10. **Second eligibility alternative can qualify.**  
    For a horizontal mismatch, if the value at `0x68` is nonzero, its paired flags need not be read. If the value at `0x50` is zero and flags at `0x54` contain `0x20`, the horizontal dependency is called.

11. **Horizontal effects can change vertical behavior.**  
    If the horizontal dependency changes the parent’s vertical endpoints to match the accumulated vertical span, the later vertical comparison may suppress notification that would have occurred against the earlier values.

12. **Both axes qualify.**  
    If both spans mismatch and both axis eligibility conditions qualify when evaluated, F-065fd89ae7c4 is called first, followed by F-d25031d00f00.

## Uncertainties

The following remain **documented_only**:

- The actual collection representation and iterator mutation guarantees.
- The complete candidate layout and which bytes extraction initializes.
- Whether accumulation computes a union, bounds, or another transformation.
- The semantic meaning of either mask and of predicate results.
- Dependency mutation, allocation, retention, and ownership behavior.
- The effects of the axis-specific calls beyond their invocation.
- Arithmetic overflow behavior and address-overflow behavior.
- Alignment requirements, byte order, concurrency guarantees, and volatile-memory semantics.
- Behavior after invalid addresses, malformed objects, dependency failure, or nonlocal exit.

A geometry-maintenance interpretation is plausible from the paired span calculations, but is not a substitute for these missing contracts.

## Evidence

This contract is based solely on static inspection of the supplied function representation and the supplied mapping to opaque dependency IDs.

Established evidence includes temporary-buffer sizes, 32-bit field accesses, byte offsets, masks, call arguments, branch conditions, short-circuit gates, and the ordering of traversal finalization and axis handling.

No dependency implementations were available. No runtime tests have been performed. The acceptance examples are consequences of the stated conditions, not observed executions.
