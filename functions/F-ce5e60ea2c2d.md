<!-- Trusted provenance: F-ce5e60ea2c2d; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-ce5e60ea2c2d

## Purpose

F-ce5e60ea2c2d conditionally performs an operation using a caller-supplied object, optionally checks readiness, and records successful completion in a one-byte status field.

The established behavior is:

- An initially observed set low status bit bypasses the operation and returns `1`.
- Otherwise, eligibility depends on the second and third inputs.
- An eligible invocation stores a four-byte value before calling preparation.
- A successful operation either completes immediately or enters a bounded readiness sequence.
- Readiness checking makes exactly two or three readiness calls when reached, with at most one intervening action.
- Successful fresh completion returns the operation’s original nonzero result, not necessarily `1`.
- Every represented normal-return path invokes the exit dependency.

“Preparation,” “readiness,” “intervening action,” and “completion” are explanatory labels for call roles and branch outcomes. Their underlying subsystem meanings are not established.

This contract describes static evidence. It does not establish dependency semantics, ownership arrangements, synchronization guarantees, or behavior outside normal returns.

## Interface

The function accepts five inputs and returns an integer.

| Input | Width or representation | Established use |
|---|---|---|
| Object address | Address represented by an integer input | Supplies the base for direct memory accesses and object-relative addresses passed to dependencies. |
| Readiness selector | Integer | Zero disables readiness checking; any nonzero value enables it. Also participates in eligibility. |
| Recorded value | 32-bit integer | Participates in eligibility and is written to object offset `0x44` on eligible paths. |
| First operation argument | Opaque 32-bit value | Passed unchanged to F-ce2b98551acc. |
| Second operation argument | Opaque 32-bit value | Passed unchanged to F-ce2b98551acc. |

The supplied representation supports four-byte integers and four-byte opaque values in the described calls and accesses. It represents addresses through an integer input, consistent with a 32-bit address model. It does not independently establish a complete ABI, address-space interpretation, pointer-provenance rules, or behavior for overflowing address calculations.

The readiness selector is tested only for zero or nonzero. Its magnitude is not a retry count. A negative selector enables checking just as a positive selector does.

### Object-relative layout

| Offset | Established access or use |
|---|---|
| `0x04` | Address passed to F-194cdec98e23. No direct field read or write at this offset is expressed by the wrapper. |
| `0x08` | Address supplied to the entry and exit dependencies. |
| `0x44` | Four-byte recorded-value field: written on eligible paths and read if the intervening action is reached. |
| `0x48` | Address supplied to F-68512f6c43c0. No direct field read or write at this offset is expressed by the wrapper. |
| `0x4c` | One-byte status field: bit zero is initially tested; successful fresh completion writes the complete byte as `1`. |

These offsets do not establish semantic subobject boundaries or dependency storage extents. A dependency receiving one of these addresses may access storage beyond that location; its contract is unavailable.

Defined execution requires valid, suitably accessible storage for every direct access actually reached, representable object-relative addresses, and satisfaction of the dependencies’ own requirements. The wrapper provides no explicit null, extent, alignment, or argument validation. Exact alignment and overall lifetime requirements remain unknown.

### Return values

On normal return, the function produces:

- `1` if the status byte observed after entry has bit zero set.
- `0` if eligibility fails, preparation returns zero, the operation returns zero, or the decisive readiness result is zero.
- The exact nonzero integer returned by F-ce2b98551acc if fresh completion succeeds.

A negative operation result is treated as nonzero success by this wrapper. If completion succeeds, that negative result is returned unchanged.

## Behavior

### Entry and status observation

The function forms the address at object offset `0x08` and calls D-01 with that address. The same computed address is used for D-02 on exit.

Only after D-01 returns does the wrapper read the one-byte status field at offset `0x4c`. It tests bit zero, ignoring the remaining bits for this decision.

If bit zero is set, the function calls D-02 and returns `1`. On this path it performs no direct recorded-value or status write and calls none of the mapped dependencies.

Consequently, the relevant status is the value after D-01, not necessarily the value present when the function was invoked. For example, an observed byte of `0x81` takes the early-return path without being rewritten by the wrapper.

### Eligibility

If the observed status bit is clear, further work is eligible if either:

- The recorded-value input is nonzero; or
- The readiness selector is zero.

The only rejected combination is a nonzero readiness selector together with a zero recorded-value input.

On rejection, the function calls D-02 and returns zero. It performs no direct write to the recorded-value field or status byte and invokes no mapped dependency.

A zero recorded value is therefore permitted when readiness checking is disabled.

### Recording and preparation

On every eligible path, the function first writes the recorded-value input as four bytes at offset `0x44`. It then calls F-321edda8d389 with the object address.

If that dependency returns zero, the function calls D-02 and returns zero. There is no resolver, operation, readiness, or intervening-action call on this path.

The recorded-value write is not rolled back if preparation fails. Dependencies may independently modify the field, so the contract establishes the earlier write rather than an unconditional final field value.

Any nonzero preparation result permits the operation phase. The wrapper makes no further use of that result’s sign or magnitude.

### Operation

Following successful preparation, the function calls F-194cdec98e23 with the address at offset `0x04`. Its opaque four-byte result becomes the first argument to F-ce2b98551acc. The fourth and fifth inputs become the second and third arguments, unchanged.

If F-ce2b98551acc returns zero, the function calls D-02 and returns zero. It makes no readiness or intervening-action call and does not directly write the status byte.

If the operation returns nonzero, that exact integer is retained as the prospective return value.

When the readiness selector is zero, the function immediately writes the entire status byte at offset `0x4c` as `1`, calls D-02, and returns the retained operation result.

### Readiness checks

When the readiness selector is nonzero and the operation returned nonzero, the function performs a bounded readiness sequence.

Each readiness call to F-900e57cefc4b is immediately preceded, in the wrapper’s call order, by a separate call to F-194cdec98e23 with the address at offset `0x04`. F-900e57cefc4b receives that fresh opaque result and the constant `0x00040000`.

The resolver result used for the operation is not reused for readiness. Results are also not cached between readiness calls. No equality or stability of these values is required by the wrapper.

There are two possible sequence shapes.

**First readiness result is nonzero**

The function makes a second resolver call and a second readiness call. This second readiness result is decisive.

- If the second result is zero, completion fails.
- If the second result is nonzero, completion succeeds.

No intervening action occurs. Exactly two readiness calls are made.

**First readiness result is zero**

After the first readiness call returns, the function reads four bytes from offset `0x44`. It calls F-68512f6c43c0 with the address at offset `0x48` and that newly read value.

It then makes a second resolver/readiness pair, followed by a third resolver/readiness pair. The third readiness result is decisive.

- A zero third result causes failure.
- A nonzero third result permits completion.

The second readiness call occurs regardless of its eventual result, and either a zero or nonzero second result is followed by the third call. Exactly three readiness calls and one intervening-action call occur on this shape.

The value supplied to F-68512f6c43c0 need not equal the original recorded-value input. Preparation, resolution, operation, or the first readiness call may have changed the field before this read. The wrapper does not use any return value from F-68512f6c43c0 to select subsequent behavior.

The bounded sequence is established by the control flow, not by the selector’s magnitude. There is no unbounded retry path: after an initial readiness miss, only one intervening action occurs before the final two readiness calls.

### Completion and exit

A zero decisive readiness result replaces the retained operation result with zero. The wrapper makes no status-byte write, calls D-02, and returns zero.

A nonzero decisive readiness result causes a one-byte write of `1` at offset `0x4c`. The function then calls D-02 and returns the retained operation result.

This is a whole-byte replacement, not a read-modify-write that preserves higher status bits.

Every represented normal-return path invokes D-02 exactly once after D-01 has returned. The supplied body does not establish exit or cleanup behavior if a dependency fails to return normally.

## Side effects

The wrapper’s direct object-memory effects are limited to:

- Reading one status byte after D-01.
- Writing four bytes at offset `0x44` on eligible, initially inactive paths.
- Reading four bytes at offset `0x44` before the intervening action, if reached.
- Writing one byte with value `1` at offset `0x4c` upon successful fresh completion.

These effects are not transactional. Failure after the recorded-value write does not restore the previous field contents or undo dependency effects.

The wrapper does not recheck the status after its initial read. If a dependency later changes the status, that change does not cause another early-return decision. Successful fresh completion still writes the complete byte as `1`.

D-02 executes after the last direct status write. Without its contract, the wrapper’s write does not guarantee that the caller subsequently observes a status byte of `1`.

The wrapper directly expresses no allocation, deallocation, reference-count adjustment, or ownership transfer. This does **not** establish that the object or any supplied address is borrowed, that ownership is retained by the caller, or that resources remain alive after dependency calls. Overall ownership and lifetime requirements are unknown. Dependencies receive the object address or object-relative addresses; whether they retain, release, invalidate, or otherwise manage them is not established.

The entry/exit pairing suggests guarded access, but mutual exclusion, memory ordering, reentrancy safety, and deadlock behavior are not established.

## Dependencies

D-01 and D-02 are document-local opaque IDs for dependencies without supplied mapped IDs.

| Dependency | Inputs at this call site | Established use |
|---|---|---|
| D-01 | Computed address at object offset `0x08` | First call; any return value is unused. Precedes the initial status read. |
| D-02 | Same computed address at object offset `0x08` | Called once on every represented normal-return path; any return value is unused. |
| F-321edda8d389 | Object address | Called after the recorded-value write. Zero blocks the operation; nonzero permits it. |
| F-194cdec98e23 | Address at object offset `0x04` | Supplies a fresh opaque four-byte argument for each operation or readiness call. |
| F-ce2b98551acc | Resolver result and the two opaque operation inputs | Zero causes failure; a nonzero result is retained unless decisive readiness fails. |
| F-900e57cefc4b | Fresh resolver result and `0x00040000` | Readiness result is interpreted as zero or nonzero. Called two or three times when readiness is reached. |
| F-68512f6c43c0 | Address at offset `0x48` and the current four-byte value read from offset `0x44` | Called exactly once if the first readiness result is zero; its result does not control the wrapper. |

No dependency is assumed to preserve object fields, global state, resource ownership, or the validity of previously supplied addresses. Normal-path descriptions are conditional on dependencies returning and on subsequent accesses remaining valid.

## Acceptance examples

These are conditional static acceptance criteria, not executed tests. Except where stated otherwise, assume dependencies return normally, all reached accesses are valid, and dependencies do not independently modify the fields discussed.

1. **Already complete.** If the status byte observed after D-01 is `0x81`, the function calls D-02 and returns `1`. No mapped dependency or direct field write occurs.

2. **Ineligible combination.** With the initial low status bit clear, a nonzero selector and zero recorded value lead to D-02 and return zero. No recorded-value write occurs.

3. **Zero value without checks.** With the initial low status bit clear and both selector and recorded value zero, the function writes four zero bytes at offset `0x44` and calls preparation.

4. **Preparation failure.** On an eligible path, preparation returning zero causes return zero after D-02. The recorded-value write precedes that failure; no operation occurs.

5. **Operation failure.** If preparation succeeds but the operation returns zero, there is no readiness sequence and no direct status write. The function returns zero after D-02.

6. **Unchecked non-Boolean success.** With checking disabled and operation result `7`, the wrapper writes status byte `1` and returns `7` after D-02.

7. **Negative operation result.** With checking disabled and operation result `-3`, the wrapper treats the result as nonzero, writes status byte `1`, and returns `-3`.

8. **First readiness success is insufficient.** Readiness results nonzero, then zero cause failure. Exactly two readiness calls occur, no intervening action occurs, and no direct status write occurs.

9. **Two-call success.** Two nonzero readiness results cause the status write and return of the retained operation result.

10. **Initial miss followed by final success.** Readiness results zero, zero, then nonzero cause one intervening action between the first and second readiness calls, followed by successful completion.

11. **Intermediate success is not decisive.** Readiness results zero, nonzero, then zero cause failure. The third call still occurs and controls completion.

12. **Fresh field value for the action.** If a preceding dependency changes offset `0x44` before the first readiness call returns zero, the intervening action receives the value subsequently read from that field, not an assumed retained copy of the input.

13. **Higher status bits are replaced.** With initial status `0x80`, successful fresh completion writes `0x01`, not `0x81`.

14. **Exit effects remain separate.** If D-02 changes the status after a successful status write, the established wrapper behavior remains the write of `1` followed by D-02. The caller-visible final byte depends on D-02.

## Uncertainties

The following remain **documented_only**:

- Exact ABI, address representation, byte order, alignment requirements, and behavior of invalid or unrepresentable address calculations.
- Semantic meaning and storage requirements of the object-relative addresses passed to dependencies.
- Meaning of the recorded value, operation arguments, and readiness constant.
- Whether resolver results represent pointers, handles, identifiers, or other four-byte tokens.
- Overall ownership and lifetime requirements, including dependency retention, release, allocation, deallocation, and invalidation behavior.
- Dependency effects on globals, shared state, and object fields.
- Blocking, exceptional exits, termination, reentry, and failure mechanisms other than ordinary return values.
- Synchronization guarantees and persistence of writes after dependency execution.

Preparation, readiness, signaling, waiting, and guarded-access interpretations are inferred intent only. In particular, no duration, unit, scheduling effect, or resource-management contract is established for the intervening action.

## Evidence

The supplied function representation and dependency-ID correspondence establish the direct access widths and offsets, zero/nonzero conditions, dependency arguments, normal-path ordering, bounded readiness sequence, whole-byte completion write, and return-value distinctions described here.

The entry/exit address is computed before entry and reused on exit. The readiness control permits only an initial check followed by confirmation, or an initial miss followed by one action and two further checks. No integer counter grows without bound in that sequence.

Ownership terminology is deliberately limited to what is directly expressed: the wrapper contains no explicit allocation, deallocation, reference-count adjustment, or ownership-transfer operation. That absence does not establish borrowing or an overall lifetime contract.

No runtime tests have been performed. Acceptance examples are derived from static control flow, not observed execution.
