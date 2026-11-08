<!-- Trusted provenance: F-c1d186dbeef6; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-c1d186dbeef6

## Purpose

F-c1d186dbeef6 repeatedly processes an indexed pair of signed 16-bit endpoints held within a mutable state object. Each iteration prepares scaled endpoint fields and invokes F-1afecaa934aa. Depending on that dependency’s result and the state it leaves behind, the function:

- Returns a fixed success indicator.
- Optionally invokes F-91f805664874 and returns a state-held result.
- Moves to the preceding endpoint pair.
- Splits the current pair into two adjacent pairs and processes the newly created pair.
- Returns the fixed value 20 when a requested split fails either of two explicit checks.

The interpretation as interval processing and subdivision is inferred from the endpoint arithmetic and overlapping pair writes. The concrete contract is the state access, call, branching, and return behavior described below. The application-level meanings of the endpoints, scale, status values, and dependency results are not established.

## Interface

### Inputs

The function accepts two arguments:

1. **State address:** An address used to access a mutable object through fixed byte offsets. The function performs no null check, bounds check, or object validation.
2. **Auxiliary value:** An opaque 32-bit value passed unchanged to F-1afecaa934aa on every iteration. The visible function does not otherwise interpret it.

The state address itself is reused throughout the invocation. Changes to the object’s fields, including changes made by dependencies, can affect subsequent processing.

### Return value

The return value is a 32-bit word. The directly established return cases are:

| Condition | Returned value |
|---|---:|
| Index is negative at an iteration boundary | 0 |
| F-1afecaa934aa returns nonzero and the resulting status is not 98 | 1 |
| F-91f805664874 is called and returns nonzero | Current 32-bit status field |
| A requested split fails its explicit checks | 20 |

These values do not establish a uniform Boolean or error-code interface. In particular, the return following F-91f805664874 is read from the state, not taken from that dependency’s return value.

### State layout

Offsets are byte offsets relative to the state address.

| Offset | Width | Interpretation in this function |
|---|---:|---|
| `0x004` | 32 bits | Signed scale used in endpoint multiplication |
| `0x020` | 32 bits | Opaque source word copied during preparation |
| `0x02C` | 32 bits | Destination of the preparation copy |
| `0x030` | 32 bits | Status/result word |
| `0x050` | 32 bits | Scaled lower endpoint |
| `0x054` | 32 bits | Scaled upper endpoint |
| `0x060` | 32 bits | Zero/nonzero gate for F-91f805664874 |
| `0x3C4 + 4 × index` | 16 bits | Signed lower endpoint of the indexed pair |
| `0x3C6 + 4 × index` | 16 bits | Signed upper endpoint of the indexed pair |
| `0x404` | 32 bits | Signed current index |

Each endpoint pair occupies four bytes. The fields at `0x3C8 + 4 × index` and `0x3CA + 4 × index` are therefore the next pair’s lower and upper endpoints.

The excerpt does not declare the object’s size, alignment, allocation, or endpoint-array capacity. Offsets describe actual accesses, not independently validated subobjects. A sufficiently large index can address unrelated state fields or inaccessible memory.

### Preconditions

For every path actually taken:

- The state object must remain accessible for all reads and writes performed by this function and its dependencies.
- Endpoint addresses derived from the current index must be valid for the corresponding 16-bit accesses.
- Relevant 32-bit fields must support the accesses shown above.
- Dependencies must satisfy whatever additional requirements they impose; those requirements are not supplied.
- Endpoint scaling and index/address arithmetic must remain within their supported numeric domain if behavior is to be determined solely from the supplied representation.

There is no established requirement that the lower endpoint be less than or equal to the upper endpoint. The function can encounter reversed pairs, and its split check handles some such cases.

## Behavior

### 1. Iteration entry

At the beginning of every iteration, the function reads the signed 32-bit index at `0x404`.

If the index is negative, it immediately returns 0. That iteration performs no preparation writes and invokes neither dependency. Writes or calls from earlier iterations are not undone.

Otherwise, the function derives the address of the indexed endpoint pair. This initial pair address is reused for both preparation endpoint reads; it is not recalculated between the two scaled-field assignments.

### 2. Preparation

The following assignments occur in order:

1. Read the indexed signed 16-bit upper endpoint, sign-extend it, multiply it by the signed 32-bit scale, and write the 32-bit result to `0x054`.
2. Read the indexed signed 16-bit lower endpoint, sign-extend it, multiply it by the scale, and write the 32-bit result to `0x050`.
3. Copy the 32-bit word at `0x020` to `0x02C`.
4. Write zero to the status field at `0x030`.

The scale is accessed separately for the two multiplication assignments; the excerpt does not establish a single captured scale value shared by both.

These are ordered assignments, not an atomic update. The excerpt does not establish the relative evaluation order of the source operands within an individual multiplication.

After preparation, the function calls F-1afecaa934aa with the state address and the unchanged auxiliary value.

### 3. F-1afecaa934aa returns zero

After a zero dependency result, the function reads the gate at `0x060`.

- If the gate is zero, F-91f805664874 is not called.
- If the gate is nonzero, F-91f805664874 is called with the state address.

If F-91f805664874 is called and returns nonzero, F-c1d186dbeef6 immediately reads and returns the current status word at `0x030`. It does not decrement the index on that path.

Otherwise, the function reads the then-current index, subtracts one, writes the result to `0x404`, and starts another iteration.

This decrement uses the index present after all applicable dependency calls, not necessarily the index used for preparation. The visible function does not preserve an earlier index across those calls for this purpose.

A zero result from F-1afecaa934aa does not itself return 0. It selects the optional-call/decrement path.

### 4. F-1afecaa934aa returns nonzero

The function reads the status field.

If status is anything other than 98, the function immediately returns 1. It does not return that status value and does not explicitly change the index on this path.

If status equals 98, the function clears status to zero and attempts a split.

The split uses a newly read current index and a newly derived endpoint-pair address. Consequently, it can operate on a different pair from the one used during preparation if F-1afecaa934aa changed the index.

### 5. Split arithmetic and rejection

For the pair selected after the dependency call, let the signed 16-bit endpoints be the lower and upper values. Both are sign-extended before arithmetic.

The midpoint is their arithmetic mean rounded toward zero. This rounding matters for negative odd sums:

- Endpoints −4 and −1 produce midpoint −2.
- Endpoints −3 and −1 produce midpoint −2.
- Endpoints 0 and 3 produce midpoint 1.

The supplied arithmetic reaches that result through a sign correction, multiplication by 32,768, and extraction of the high half. Its intermediate values fit signed 32-bit range for all signed 16-bit endpoint combinations:

- Endpoint sum: −65,536 through 65,534.
- Sign-corrected sum: −65,535 through 65,534.
- Product: −2,147,450,880 through 2,147,418,112.

Thus the midpoint calculation itself does not require overflowing signed 32-bit arithmetic. The right-shift operation has the arithmetic, sign-preserving interpretation represented by the supplied data.

The split is rejected if either:

- The current signed index is greater than 6; or
- The midpoint is less than the signed lower endpoint.

On rejection, the function performs these writes in order:

1. Write 20 to status.
2. Write 0 to the index.
3. Return 20.

The split’s upper-index check occurs only after preparation, the dependency call, the status decision, and midpoint calculation. It is not an entry bounds check and cannot protect earlier endpoint accesses.

The split path does not explicitly reject a negative index introduced by a dependency before calculating its endpoint address.

### 6. Successful split

If neither rejection condition holds, the function performs these writes in order:

1. Write the midpoint as a 16-bit value into the next pair’s lower endpoint.
2. Copy the current pair’s upper endpoint into the next pair’s upper endpoint.
3. Write midpoint minus one into the current pair’s upper endpoint.
4. Read the current index, add one, and write the resulting 32-bit value to `0x404`.

The current pair’s lower endpoint is not explicitly changed.

Under ordinary, non-overlapping storage conditions, an original pair with endpoints lower and upper becomes:

- Current pair: lower through midpoint minus one.
- Next pair: midpoint through upper.

The function then starts another iteration, normally processing the next pair because the index was incremented.

The midpoint-minus-one value fits signed 16 bits except when the midpoint is −32,768. That boundary produces −32,769 before assignment to the 16-bit field. The represented 16-bit write has low bits `0x7FFF`, interpreted as 32,767 by later signed endpoint reads. The split checks do not exclude this boundary.

The pair address used for the midpoint and first destination write is retained. The index is read again when forming the subsequent destination addresses and the final increment. No dependency call occurs between these writes, but the excerpt does not establish safety against overlapping state fields or concurrent mutation.

## Side effects

The function mutates the caller-supplied state in place.

Direct writes can affect:

- Both scaled endpoint fields.
- The preparation-copy destination.
- Status.
- The index.
- Three 16-bit endpoint fields during a successful split.

There is no rollback on any return path. Preparation changes remain visible even when the next dependency result causes an immediate return.

The visible body contains no allocation, release, ownership transfer, synchronization, or external I/O operation. This does not establish that its dependencies lack those effects.

The state address is passed directly to both dependencies. No copy isolates their changes. Their modifications can affect later status checks, gate decisions, endpoint selection, index movement, and return values.

Termination is not guaranteed by the visible loop alone. Dependencies may alter the state, and successful splits can be interleaved with decrements.

## Dependencies

### F-1afecaa934aa

- **Arguments:** State address and opaque 32-bit auxiliary value.
- **Invocation:** Once per iteration whose entry index is nonnegative, after all preparation writes.
- **Result interpretation:** Zero selects the optional-call/decrement path; any nonzero value selects the status-check path.
- **Unknowns:** Full input contract, state mutations, global effects, memory lifetime effects, and application-level meaning of its return value.

The function does not assume that this dependency preserves the current index, endpoints, gate, or status. A nonzero return is not by itself sufficient to request splitting; status must also equal 98.

### F-91f805664874

- **Arguments:** State address.
- **Invocation:** Only after F-1afecaa934aa returns zero and the subsequently read gate is nonzero.
- **Result interpretation:** Nonzero causes an immediate return of the state’s status word. Zero allows index decrement and continued iteration.
- **Unknowns:** Full input contract, state mutations, global effects, memory lifetime effects, and meaning of its return value.

Neither dependency’s exact nonzero return magnitude is otherwise used.

## Acceptance examples

These are conditional acceptance examples derived from static evidence. They are not executed tests. Except where stated otherwise, they assume valid, non-overlapping storage, no concurrent mutation, representable arithmetic, and dependencies that preserve fields not explicitly changed in the example.

1. **Already exhausted**

   If the entry index is −1, the function returns 0 without preparing fields or calling either dependency.

2. **Preparation followed by fixed success**

   With index 0, scale 3, and endpoints 2 and 5, preparation writes 15 to the scaled upper field before writing 6 to the scaled lower field. It then performs the preparation copy and clears status.

   If F-1afecaa934aa returns nonzero and leaves status 7, the function returns 1, not 7.

3. **One decrement reaches exhaustion**

   With index 0, suppose F-1afecaa934aa returns zero, leaves the index unchanged, and leaves the gate zero. The function decrements the index to −1. At the next iteration entry it returns 0. F-91f805664874 is not called.

4. **Optional dependency supplies the returned status**

   Suppose F-1afecaa934aa returns zero and the gate is nonzero. If F-91f805664874 writes status 42 and returns nonzero, the function returns 42 without decrementing the index.

   If that dependency instead leaves status zero while returning nonzero, the outer function returns 0.

5. **Ordinary split**

   Suppose the post-call index is 2, its endpoints are 10 and 15, F-1afecaa934aa returned nonzero, and status is 98.

   The midpoint is 12. Status is cleared. The next pair becomes 12 through 15; the current pair becomes 10 through 11. The index becomes 3, and processing continues rather than returning immediately.

6. **Index-limit rejection**

   Suppose a split is requested with post-call index 7 and valid endpoint storage. Regardless of endpoint ordering, the index check rejects the split. Status becomes 20, index becomes 0, and the function returns 20.

7. **Endpoint-order rejection**

   For endpoints 9 and 2, the midpoint is 5. A requested split at an otherwise permitted index is rejected because 5 is less than 9. Status becomes 20, index becomes 0, and the return is 20.

8. **Minimum-endpoint boundary**

   For endpoints −32,768 and −32,768 at a permitted index, the midpoint equals the lower endpoint, so the midpoint comparison does not reject the split. The next pair receives those same endpoints. The current upper endpoint receives the 16-bit representation `0x7FFF` from midpoint minus one.

9. **Dependency-selected split pair**

   If preparation used index 1 but F-1afecaa934aa changes the index to 4, returns nonzero, and sets status to 98, the split uses pair 4. The already-written scaled fields still reflect the preparation accesses for pair 1.

## Uncertainties

The following are **documented_only**, not assumed behavior:

- The object’s declared type, total size, alignment, and endpoint capacity.
- The meanings of status 98, status 20, the gate, and the copied word.
- Whether the index is intended to represent a stack position, search depth, or another traversal state.
- Dependency ownership, allocation, release, global-state, and exception or fault behavior.
- Behavior of scale multiplication outside signed 32-bit range.
- Behavior of overflowing index arithmetic or invalid address calculations.
- Guarantees concerning concurrent access, volatile storage, or memory-mapped fields.
- Application-level validity requirements for endpoint ordering.
- Whether higher-level invariants make the minimum-endpoint split boundary unreachable.

The cutoff at index 6 does not establish an endpoint-array capacity. Conversely, the absence of an early upper bound does not establish that arbitrary nonnegative indices are supported.

## Evidence

**Established static evidence:** The supplied representation directly supports the field widths and offsets, ordered assignments, dependency argument lists, zero/nonzero branching, status constants, index updates, split writes, and return cases described above.

**Derived arithmetic evidence:** Signed 16-bit endpoint bounds establish the stated midpoint intermediate ranges. Under the represented arithmetic right-shift semantics, the midpoint equals the endpoint mean rounded toward zero.

**Inferred intent:** Endpoint pairs appear to form a mutable interval worklist, with successful splitting followed by processing of the next pair. This interpretation does not add guarantees about ordering, completeness, progress, or termination.

**Runtime evidence:** None. No runtime tests have been performed.
