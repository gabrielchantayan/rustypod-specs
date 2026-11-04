<!-- Trusted provenance: F-daf78a556d2c; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-daf78a556d2c

## Purpose

F-daf78a556d2c conditionally invokes a request dependency using an object-associated 32-bit field and a writable local value. If the request returns nonzero, the function publishes the low byte of that local value through a global-derived destination and obtains its pending return value from another dependency.

An unsigned state byte controls whether an additional ordered pair of calls occurs. That decision is not equivalent to request success or failure: the pair can run without a request, after a zero request result, or after a nonzero request result.

These are established control-flow and data-flow properties. “Request” and “publication” describe the visible operations, not verified application semantics. The meanings of the object fields, numeric arguments, and return values 11 and 28 are not established.

## Interface

### Arguments and directly accessed layout

The function takes two arguments:

| Argument | Visible representation | Use |
|---|---|---|
| Object address | 32-bit address-like value | Base for two field accesses; also passed unchanged to dependencies |
| Initial value | Uninterpreted 32-bit value | Initializes a writable local working cell |

The directly accessed object fields are:

| Offset from object address | Width | Use |
|---|---:|---|
| 0x2D4 | 32 bits | Compared with zero; conditionally read again for the request argument |
| 0x2E3 | 8 bits, unsigned | Compared with the boundary between 3 and 4 |

**Every normally completing execution reads both fields. Both fields are read before the first dependency invocation, if any.** Their read counts and timing depend on the path:

- An initial zero handle-like field leads to a state-byte read without a request.
- An initial nonzero handle-like field leads immediately to a state-byte read.
- A request path includes another handle-field read before the request and another state-byte read after the request-related operations.

The function does not validate the object address. Normal execution requires readable storage for both fields, not merely whichever field participates in a particular branch. The known layout reaches through byte offset 0x2E3; this does not establish a complete object layout or allocation size.

### Return value

The returned value is 32 bits. On normal completion it is selected as follows:

| Condition | Pending return value |
|---|---|
| No request occurs | 11 |
| F-cd7e1905512f returns zero | 28 |
| F-cd7e1905512f returns nonzero | The complete 32-bit result of F-46e5cb3247c2 |

The additional operation pair does not replace this pending value.

There is no caller-supplied output pointer. The pointer given to the request dependency addresses the function’s local 32-bit working cell, initialized from the second argument. It does not identify the caller’s original argument storage.

### Preconditions and representation limits

Normal execution requires:

- Readable storage for both object fields, including any repeated reads on the selected path.
- Valid arguments and execution conditions for every dependency invoked.
- A writable byte at offset 3 from the global-derived destination base if publication occurs.
- A valid lifetime for the object through all operations that use it.

The supplied evidence does not define alignment requirements, synchronization requirements, invalid-address behavior, or dependency-specific preconditions.

The visible representation uses 32-bit values and address-like operands. It does not establish a portable rule for overflowing address arithmetic. The field addresses and publication destination must therefore be valid without relying on an assumed wrapping-address convention.

## Behavior

### Initialization and first decision

The function initializes its pending return value to 11 and its local 32-bit working cell to the second argument.

It then reads the 32-bit field at object offset 0x2D4 and tests whether it is zero.

If the field is zero, the function skips the request and publication operations and reads the unsigned state byte at offset 0x2E3:

- A value from 0 through 3 causes a return of 11 with no dependency calls.
- A value from 4 through 255 selects the additional operation pair, after which the function returns 11.

If the initial field is nonzero, the function reads the unsigned state byte before invoking any dependency:

- A value from 4 through 255 immediately selects the additional operation pair. The function then returns 11.
- A value from 0 through 3 allows the request path to proceed.

Thus neither a missing handle-like value nor an immediately disqualifying state avoids reading either object field.

### Request arguments and result classification

On the request path, the function reads the 32-bit field at offset 0x2D4 again. It invokes F-cd7e1905512f with these seven arguments, in order:

1. The value from the second read of the handle-like field.
2. Numeric value 0.
3. Numeric value 8.
4. Numeric value 6.
5. The address of the local 32-bit working cell.
6. Numeric value 1.
7. The original object address.

The initial handle test and the request argument use distinct reads. The function does not retest the second value for nonzeroness. If the execution environment validly permits a change between reads, the request receives the newer value even if that value is zero.

The local working cell initially contains the second argument. F-cd7e1905512f receives its address and may modify it. The argument value 1 does not establish a byte count, element count, access width, or direction of transfer.

The request’s 32-bit result is classified only as zero or nonzero:

- Zero changes the pending return value to 28 and skips publication and F-46e5cb3247c2.
- Any nonzero bit pattern selects publication.

There is no positive-only test. A value whose highest bit is set still selects the nonzero path.

### Nonzero request result and publication

For a nonzero request result, the function performs these operations in order:

1. Calls an unmapped pre-store dependency with an argument obtained from a global source.
2. Stores one byte at offset 3 from a global-derived base address.
3. Calls an unmapped post-store dependency with an argument obtained from the same global source used for the first call.
4. Calls F-46e5cb3247c2.

The stored byte is the low-order eight bits of the working cell as used by the store. For a 32-bit working value \(W\), its numerical byte value is \(W \bmod 256\). Only one destination byte is directly written; the function does not copy the complete working cell to that destination.

The first and second bracketing calls use the same global argument source, but the source is consulted separately at their respective call sites. The first call may affect the value later supplied to the second call. The publication base is obtained for the store after the pre-store call; it must not be treated as a snapshot taken before that call.

Likewise, the working-cell value used for publication must not automatically be equated with its value immediately after the request. A dependency that can access the cell could change it before the store. The visible function does not itself modify the cell between the request and publication.

The surrounding calls are not established lock or synchronization operations. Their placement does not prove atomic publication, mutual exclusion, or preservation of global values.

F-46e5cb3247c2 receives, in order:

1. The nonzero 32-bit result from F-cd7e1905512f.
2. Numeric value 0.
3. A value obtained from another global source at this stage.

Its complete 32-bit result becomes the pending return value. No special result from this dependency causes an early exit or skips the subsequent state-byte decision.

### State-byte decision after the request

After the request path, the function reads the unsigned state byte at offset 0x2E3 again. This happens after either:

- A zero request result, with pending value 28; or
- The complete publication sequence and F-46e5cb3247c2 call, with that dependency’s result pending.

The later state read controls the next action:

- Values 0 through 3 return the pending value directly.
- Values 4 through 255 select the additional operation pair before returning the pending value.

The later read is not a reuse of the earlier state value. Request or publication dependencies can therefore affect this decision through their effects on accessible state.

An initially nonzero handle-like field and an initial state of at least 4 take a different path: they select the pair immediately, without the request or a second state read.

### Additional operation pair

When selected, the function invokes:

1. F-e35b2bccb75d with the original object address, 0, and 0.
2. F-c6c589453fa8 with 8, 0, 13, and 4.

The second call occurs only after the first returns normally. Both return values are ignored.

The function then returns the previously selected pending value. There is no further direct state-byte read after the pair, even if either dependency changes that byte.

The pair’s application purpose is unknown. Cleanup, notification, or state transition must not be treated as established behavior.

## Side effects

### Direct reads and writes

The following table captures the direct object-field reads on each normally completing path:

| Path | Handle-field reads | State-byte reads | Ordering before dependencies |
|---|---:|---:|---|
| Initial handle zero | 1 | 1 | Handle, then state |
| Initial handle nonzero; initial state at least 4 | 1 | 1 | Handle, then state |
| Initial handle nonzero; initial state below 4 | 2 | 2 | Handle, state, second handle, then request; second state follows request-related operations |

The function also directly initializes its local working cell. On the nonzero request path, it directly writes exactly one byte through a global-derived destination.

There is no direct visible assignment to either object field. That does not guarantee those fields remain unchanged: dependencies receive the object address or may have other access to shared storage.

No disjointness guarantee is established. The publication destination could alias object storage or other storage observed later. Such aliasing can affect the later state read if the addresses coincide.

### Dependency effects and abnormal completion

Dependency effects are not bounded by the wrapper’s visible operations. In particular, the evidence does not establish that any dependency:

- Preserves object fields or global sources.
- Leaves the working cell unchanged.
- Allocates, retains, consumes, or releases a resource.
- Returns normally.
- Provides synchronization or memory-ordering guarantees.

A fault or nonreturning dependency can prevent later operations and the final return. The visible body contains no explicit recovery or error-translation mechanism for such outcomes.

### Ownership and lifetime

No allocation or deallocation is directly identifiable in this function.

The nonzero request result is passed to F-46e5cb3247c2, but that forwarding does not establish ownership transfer or release. There is no separately visible release operation whose contract can be established here.

The working cell is local to this invocation. Its address is supplied to F-cd7e1905512f. Whether that dependency retains the address is unknown; the function does not establish a valid lifetime for using the cell after this invocation ends.

## Dependencies

| Dependency | Visible use | Result handling |
|---|---|---|
| F-cd7e1905512f | Seven-argument request using the reread handle-like field, working-cell address, and object address | Zero selects 28; nonzero selects publication and is later forwarded |
| F-46e5cb3247c2 | Receives the nonzero request result, 0, and a global-derived value after publication | Full 32-bit result becomes pending return value |
| F-e35b2bccb75d | First call in the state-selected pair; arguments: object address, 0, 0 | Ignored |
| F-c6c589453fa8 | Second call in that pair; arguments: 8, 0, 13, 4 | Ignored |
| Unmapped pre-store dependency | Receives a global-derived argument before the byte store | Ignored |
| Unmapped post-store dependency | Receives a separately obtained argument from the same global source after the store | Ignored |

Opaque IDs for the two store-bracketing dependencies were not supplied. Their identities remain documented_only; assigning invented IDs or synchronization contracts would exceed the evidence.

## Acceptance examples

These are conditional consequences of the static evidence, not executed tests. Unless otherwise stated, each assumes valid memory and normal return from invoked dependencies.

1. **Zero handle and low state.**  
   The initial handle-field read yields zero, and the state-byte read yields 3. Both fields are read, in that order. No dependency is invoked, no global byte is written, and the function returns 11.

2. **Zero handle and threshold state.**  
   The initial handle-field read yields zero, and the state-byte read yields 4. Both reads precede F-e35b2bccb75d, which precedes F-c6c589453fa8. The function returns 11 without making a request.

3. **Nonzero handle and high initial state.**  
   The first handle value is nonzero and the first state value is 255. Each object field is read once. The additional pair runs; the request and publication sequence do not. The result is 11.

4. **Zero request result with low later state.**  
   The initial handle is nonzero and the initial state is 0. The handle is reread for F-cd7e1905512f, which returns zero. The later state is 3. No publication occurs, the additional pair is skipped, and the result is 28.

5. **Zero request result with changed state.**  
   The request is permitted by an initial state below 4 and returns zero. The later state read yields 4. The additional pair runs, and the result remains 28.

6. **Nonzero request result and byte truncation.**  
   The request returns nonzero and leaves the working cell equal to 0x1234ABCD. If no intervening effect changes that cell, the publication writes 0xCD. If F-46e5cb3247c2 returns 0x80000005 and the later state is below 4, that complete 32-bit value is returned.

7. **Additional pair preserves result selection.**  
   After publication, F-46e5cb3247c2 returns zero and the later state is 4. The additional pair runs. Its return values do not replace the pending zero; the function returns zero, not 11 or 28.

8. **Different values at the two handle reads.**  
   If valid environmental behavior permits the first handle read to be nonzero and the second to be zero, with an initial state below 4, F-cd7e1905512f receives zero as its first argument. There is no second nonzero gate.

9. **Changed global argument source.**  
   If the pre-store dependency changes the global source used for its argument, the post-store dependency receives the value obtained from that source afterward. Equal arguments are not required.

## Uncertainties

The following remain documented_only:

- Application meanings of the fields, numeric constants, and return values.
- Exact dependency types beyond the visible representations.
- Identities and contracts of the two unmapped dependencies.
- Dependency access widths through the working-cell pointer.
- Global layout, aliasing, initialization, and lifetime requirements.
- Ownership and release behavior for the request result.
- Valid concurrency rules for changes between direct reads.
- Fault, exception, invalid-memory, and address-overflow behavior.

An apparent request, publication, or finalization role is inferred intent, not an established external API contract.

## Evidence

This contract is derived from the supplied static function body and opaque dependency mapping. The body establishes field offsets and access widths, unsigned state comparisons, repeated reads, argument values and order, the single-byte store, dependency-call ordering, and return-value selection.

In particular, both branches of the initial handle decision reach a state-byte read before any dependency: the zero branch reaches the later check, while the nonzero branch reaches the earlier check. Read counts differ by path; the requirement to read both fields does not.

No dependency implementations, runtime observations, external API definitions, or complete memory-layout documentation were supplied. No runtime tests have been performed.
