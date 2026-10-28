<!-- Trusted provenance: F-980bade46bea; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-980bade46bea

## Purpose

F-980bade46bea conditionally obtains candidate values from an object-associated callback and returns the first candidate that passes a metadata-byte filter, unless the object’s observed state terminates processing first.

The established behavior has three stages:

1. Evaluate an entry gate using F-23ff01708a12.
2. If the gate permits processing, invoke the object-associated callback repeatedly.
3. Return either an accepted nonzero candidate or zero when the observed state equals 3.

After an unsuccessful iteration, F-d8ab044b167a is called before another callback invocation. Its purpose is not established. In particular, the available evidence does not establish that it advances an iterator, releases a candidate, or guarantees progress.

Interpreting this routine as candidate enumeration with a terminal-state marker is inferred intent. The gate, callback, filter, state observations, and return conditions below are established static behavior.

## Interface

**Input:** one pointer to a readable object-like storage region.

**Output:** one integer result:

- Zero if the entry gate returns zero.
- Zero if a state observation following a callback finds state 3.
- Zero if a later state observation, after a candidate passes the metadata-byte test, finds state 3.
- Otherwise, the nonzero candidate returned by the callback that passed both relevant checks.

The supplied representation uses integer-sized object slots and integer-valued dependency results. Under its apparent 32-bit data model, the following layout applies:

| Location relative to the input object | Interpretation used by this function |
|---|---|
| Offset 0x00 | Address of a dispatch-table-like region |
| Offset 0x04 | Integer argument supplied to F-23ff01708a12 |
| Offset 0x30 | Integer slot whose low-order byte is compared with 3 |

The callback target is obtained from offset 0x48 within the dispatch-table-like region. That offset is relative to the region addressed by the first object slot, not relative to the object itself.

F-418cc7cf316c returns an address-like integer. A single unsigned byte at offset 0x74 from that address is used for filtering.

The precise binary ABI, pointer width, calling convention, and signed integer representation are not independently established. The concrete object offsets above rely on the apparent four-byte integer layout. The dispatch and metadata displacements, 0x48 and 0x74, are explicit byte displacements in the supplied representation.

### Preconditions

For a defined execution:

- The input must identify storage valid for each object read actually reached.
- The object must remain valid across dependency calls and callback invocations whenever subsequent reads are reached.
- If the entry gate is nonzero, the first object slot must provide a valid dispatch-region address, and the entry at displacement 0x48 must designate a compatible callable target.
- If a nonzero candidate reaches metadata filtering, the address returned by F-418cc7cf316c must permit reading the byte at displacement 0x74.
- Any address arithmetic and indirect call must be valid under the actual execution environment.

The function performs no explicit null checks, dispatch-target validation, metadata-address validation, or recovery from invalid memory access.

## Behavior

### Entry gate

The function reads the object’s second integer slot and passes its value to F-23ff01708a12.

If that dependency returns zero, the function returns zero immediately. On this path, it does not perform its own dispatch lookup, callback invocation, state comparison, metadata lookup, or F-d8ab044b167a call.

A nonzero gate result permits processing but is not itself a candidate and is not returned as the successful result. Its sign and magnitude do not otherwise affect the visible control flow.

The state slot is not checked before this gate call or before the first callback. Therefore, a state already equal to 3 at entry does not by itself prevent either of those calls.

### Callback invocation and first state observation

Each iteration begins by resolving the object-associated callback through the first object slot and dispatch displacement 0x48. The callback receives the original object pointer and returns an integer candidate.

Dispatch resolution occurs again on each iteration. The function does not explicitly retain a callback target across iterations. A dependency that changes the first object slot or the dispatch entry can therefore affect a later invocation, provided all subsequent accesses remain valid.

After the callback returns, the function observes the low-order byte of the state slot.

- If that value is 3, the function returns zero.
- Otherwise, the callback result is considered for filtering.

The callback precedes this state observation even when the state was already 3 before the iteration began. A callback can change the state before the observation; the observed post-callback value controls this decision.

When this observation finds 3, a nonzero callback result is discarded without metadata filtering. F-d8ab044b167a is not called on that exit path.

### Candidate filtering

A zero candidate fails without a call to F-418cc7cf316c.

For a nonzero candidate, the function calls F-418cc7cf316c with that candidate. It then reads the unsigned byte at displacement 0x74 from the returned address.

The filter passes exactly for byte values 0 through 4. Values 5 through 255 fail.

This is an unsigned one-byte comparison. A stored byte of 255 does not pass as though it were a negative signed byte. A candidate itself need only be nonzero to reach this dependency; no positive-only restriction is visible.

The metadata result does not replace the candidate as the function’s return value. It supplies the address for the byte test.

### Second state observation and successful return

If the candidate is nonzero and its metadata byte is below 5, the function observes the state slot again.

- If the low-order byte is now 3, it returns zero.
- Otherwise, it returns the candidate produced by the callback.

This second observation matters because F-418cc7cf316c may change the object or other reachable state. No preservation guarantee for that dependency is established.

The second observation is conditional: it occurs only after a candidate passes the metadata-byte filter. A failed byte test does not trigger this additional state check before F-d8ab044b167a.

The return value on success is the saved callback result, not a subsequently read object field and not the metadata dependency’s result.

### Unsuccessful iteration

If the first state observation was not 3, but either:

- the candidate was zero, or
- the candidate’s metadata byte was at least 5,

the function calls F-d8ab044b167a with the original object pointer. Its return value, if any, does not control this function’s visible decisions.

The next iteration then begins with another callback invocation. There is no state check between F-d8ab044b167a and that invocation.

Consequently:

- If F-d8ab044b167a sets the state to 3, the next callback still runs before the state is checked.
- If F-418cc7cf316c sets the state to 3 but the metadata byte fails the filter, F-d8ab044b167a still runs before any further state observation by this function.
- If F-d8ab044b167a or the next callback changes the state away from 3, the next post-callback observation can permit continued processing.

There is no explicit iteration bound. If candidates never pass and the post-callback state observations never equal 3, processing can continue indefinitely.

### Width and arithmetic considerations

The state comparison converts an integer slot to a character-sized value before comparison with 3. In the apparent eight-bit-byte model, only the low-order eight bits determine whether the comparison succeeds. Upper bits do not need to be zero.

For example, under the apparent 32-bit integer model, state-slot values 0x00000003 and 0x00000103 both compare as state 3. Character signedness does not change equality with the representable value 3 for this truncation.

This is a low-order-value interpretation, not an assertion that the first byte in memory is always the state byte. The corresponding byte address depends on endianness and the actual generated access.

The function contains no explicit candidate increment, count decrement, or retry counter. The visible address calculations use fixed displacements. Overflow handling and invalid-address behavior are not provided.

## Side effects

No direct object assignment or explicit memory release is visible in F-980bade46bea. Its direct memory operations are reads.

Its calls can nevertheless have arbitrary effects within their actual contracts. The supplied evidence does not establish that any dependency preserves:

- The state slot.
- The dispatch-region address or callback target.
- Other object contents.
- Global state.
- Candidate or metadata lifetime.
- The validity of the input object.

For a reached iteration, the established ordering is:

1. Resolve and invoke the callback.
2. Observe the state.
3. For a nonzero candidate only, call F-418cc7cf316c and read its metadata byte.
4. For a passing byte only, observe the state again and return.
5. Otherwise, call F-d8ab044b167a before repeating.

There is no visible cleanup when returning zero or a candidate. Ownership transfer is not established for the input, candidate, or metadata address. In particular, callers must not infer that a rejected candidate was released, that a returned candidate acquires a new reference, or that zero means all associated resources were cleaned up.

No logging, error-code translation, exception handling, synchronization, or explicit cancellation mechanism is visible. Zero merges several control-flow outcomes; it does not identify which one occurred.

## Dependencies

| Opaque dependency | Argument | Result use | Established call condition |
|---|---|---|---|
| F-23ff01708a12 | Value of the object’s second integer slot | Zero/nonzero entry gate | Once at entry |
| Object-associated indirect callback; opaque ID unavailable | Original object pointer | Integer candidate | At the start of every reached iteration |
| F-418cc7cf316c | Nonzero callback candidate | Address for the unsigned-byte read at displacement 0x74 | First post-callback state observation is not 3, and candidate is nonzero |
| F-d8ab044b167a | Original object pointer | Result not used for control flow | Candidate is zero or fails the metadata-byte test, after a nonterminal first state observation |

No dependency is assumed to be pure, to preserve globals, or to own or release memory. The callback’s identity and complete contract are documented_only.

## Acceptance examples

These examples are conditional on valid memory, compatible calls, and the stated dependency behavior. They are expected outcomes from static analysis, not executed tests.

1. **Gate denies entry.**  
   F-23ff01708a12 returns zero. The result is zero. No callback or later dependency is invoked by this function, regardless of the object’s state value.

2. **Terminal state after a productive callback.**  
   The gate is nonzero. The callback returns candidate 41 and leaves the low-order state byte at 3. The result is zero. F-418cc7cf316c is not called for candidate 41.

3. **Accepted candidate at the upper passing boundary.**  
   The callback returns 41. The first state observation is not 3. F-418cc7cf316c supplies accessible metadata with byte value 4, and the second state observation is not 3. The result is 41. F-d8ab044b167a is not called for that iteration.

4. **First failing metadata value.**  
   The callback returns a nonzero candidate and the first state observation is not 3. The metadata byte is 5. F-d8ab044b167a is called, and another callback invocation follows if that call returns normally. There is no immediate second state observation on this path.

5. **Zero candidate.**  
   The callback returns zero while the first state observation is not 3. Metadata lookup is skipped. F-d8ab044b167a runs before the next callback.

6. **Metadata lookup causes termination.**  
   The callback returns a nonzero candidate, and the first state observation is not 3. F-418cc7cf316c changes the state to 3 and supplies metadata byte 0. The second state observation causes a zero result, not return of the candidate.

7. **State changes during a failing lookup.**  
   F-418cc7cf316c changes the state to 3, but the metadata byte is 255. The function calls F-d8ab044b167a rather than returning immediately. A subsequent callback occurs before the next state observation.

8. **State already terminal at entry.**  
   The entry state is 3, but the gate is nonzero. The callback still runs. If it leaves state 3, the function returns zero; if it changes the state, subsequent behavior follows the observed new value.

9. **Upper state bits are present.**  
   Under the apparent 32-bit model, the callback leaves the state slot at 0x00000103. The low-order-byte comparison finds 3, so the result is zero.

10. **No eventual progress.**  
    Every callback returns zero, every first state observation differs from 3, and F-d8ab044b167a returns normally without enabling termination. The function does not return.

## Uncertainties

The following remain documented_only:

- The object’s concrete type and complete layout.
- The exact ABI, pointer width, endianness, and generated memory-access widths.
- The semantic meaning of state 3.
- The identities and contracts of possible indirect callback targets.
- Whether candidates represent pointers, handles, identifiers, or another integer-valued entity.
- The meaning of the metadata byte and why values below 5 are accepted.
- Whether F-d8ab044b167a advances state, disposes resources, or performs unrelated work.
- Ownership, lifetime, reference-counting, and aliasing guarantees.
- Whether dependencies mutate globals or object storage.
- Any concurrency guarantees or safety under asynchronous mutation.
- Behavior after a fault, incompatible call, invalid address, or dependency that does not return normally.

The apparent enumeration and terminal-state interpretation is inferred intent only. None of these unknowns should be resolved by assuming conventional iterator or destructor behavior.

## Evidence

This contract derives solely from static inspection of the supplied restricted representation of F-980bade46bea and the supplied opaque dependency mapping.

Established evidence includes the entry gate, indirect dispatch displacement, candidate-dependent metadata lookup, unsigned-byte threshold, two differently positioned state observations, unsuccessful-iteration call, and integer return paths.

The four-byte object-slot layout is conditional on the apparent 32-bit representation. No external type definitions, dependency implementations, binary ABI documentation, or runtime observations were supplied.

No runtime tests have been performed. The acceptance examples describe conditional behavior implied by the static evidence, not verified executions.
