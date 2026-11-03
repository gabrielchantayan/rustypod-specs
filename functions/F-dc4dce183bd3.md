<!-- Trusted provenance: F-dc4dce183bd3; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-dc4dce183bd3

## Purpose

F-dc4dce183bd3 initializes up to two caller-supplied result slots, selects a status value from shared state, and conditionally invokes F-1c61b1704268 after two calls to F-c19b4c0cb7f7.

The established behavior is a conditional dispatch gate. A null primary argument prevents both dependency calls. A nonnull primary argument causes two separate probe calls; delegation occurs only when selected portions of their respective return values are zero.

Interpreting the probes as format checks, character reads, or validation is **inferred intent**, not established behavior. The available evidence does not identify the meanings of either shared status value, either output slot, or the data addressed through the primary argument.

## Interface

### Arguments

The function accepts four arguments, in this order:

1. **Primary reference.** A nullable pointer to an integer-sized storage location. When nonnull, its stored value participates in address-like arithmetic for each probe call. The same pointer is passed to F-1c61b1704268 if delegation occurs.
2. **Requested displacement.** An integer value. Zero selects a displacement of 12 for probe argument construction. Any nonzero value is used unchanged for that construction. Delegation receives the original requested displacement, including zero.
3. **First optional output.** A nullable pointer to a 32-bit storage location.
4. **Second optional output.** A nullable pointer to a 32-bit storage location.

The supplied representation uses integer and unsigned-integer types consistent with 32-bit values and explicitly represents the output stores and function result as four-byte quantities. Exact calling-convention details and source-language signed-overflow rules are not established by this representation.

### Result

The function returns one 32-bit quantity:

- The value read from a **default-status shared location**, if the primary reference is null.
- The value read from a **probe-rejection-status shared location**, if the primary reference is nonnull but the probe predicate fails.
- The result returned by F-1c61b1704268, if the probe predicate passes and that dependency returns normally.

These are selection rules, not interpretations of success or failure. No particular numeric status value is established.

### Preconditions

For defined, normally completing execution:

- Each nonnull output pointer must permit a four-byte zero store.
- A nonnull primary reference must permit the integer reads used to construct the two probe arguments.
- The values supplied to each dependency must satisfy that dependency’s requirements, which are not available here.
- The shared status locations must be readable when their respective reads occur.

There is no established requirement that the output pointers be distinct from one another or from other reachable storage. Aliasing can affect behavior because output initialization precedes the primary-reference reads.

The function does not explicitly reject a zero value stored through a nonnull primary reference. A null primary reference and a nonnull primary reference containing zero are different cases.

## Behavior

### Output initialization and initial status read

The function first initializes the optional outputs:

1. If the first optional output is nonnull, it receives a 32-bit zero.
2. If the second optional output is nonnull, it receives a 32-bit zero.

The first store precedes the second. Both occur before the null-primary decision, any probe call, and either dependency invocation.

After these stores, the function reads the default-status shared location. This read occurs even when the primary reference is nonnull and the eventual result will come from another source.

If the primary reference is null, the function returns that captured default-status value. Neither dependency is called, and the probe-rejection-status location is not read by the visible body.

Output initialization is therefore not conditional on the availability or validity of the primary input. A null primary reference does not suppress writes to valid, nonnull output pointers.

### Effective displacement

For a nonnull primary reference, the function selects an effective displacement:

- Requested displacement zero: effective displacement is 12.
- Requested displacement nonzero: effective displacement equals the requested displacement.

This choice applies to probe argument construction only. It does not replace the requested displacement subsequently passed to F-1c61b1704268.

The effective displacement is selected before the first probe call and is reused for the second probe call.

### Two ordered probe calls

The function calls F-c19b4c0cb7f7 twice, in sequence.

For each call, its argument is formed from:

- The integer currently read through the primary reference;
- The effective displacement;
- An additional displacement of 14.

The addition is integer arithmetic, not scaling by the size of the pointed-to integer. No alignment adjustment, range validation, or bounds check is visible.

The integer stored through the primary reference is read separately for each call. Consequently, the two dependency arguments need not be equal. For example, a side effect of the first call could change the stored value used by the second call.

The first probe result is reduced to a one-byte character-sized value. Only whether that byte equals zero matters. Its signed interpretation does not affect this equality test.

The second probe result is treated as an unsigned integer for a mask that selects bits 8 through 15. Bits outside that range do not affect the second part of the gate.

Both calls occur before the gate is evaluated. A nonzero byte from the first result does **not** suppress the second call. Likewise, the second call is not a substitute for, or a reuse of, the first result.

### Probe-rejection status and gate

After both probe calls return, the function reads the probe-rejection-status shared location. This read occurs before deciding whether to delegate, including when delegation will occur.

Delegation requires both conditions:

- The byte retained from the first probe result is zero.
- Bits 8 through 15 of the second probe result are all zero.

The two conditions inspect different portions of different calls’ results. They do not establish that either complete result is zero.

If either condition fails, the function returns the captured probe-rejection-status value. The visible body performs no additional output stores on this path. However, dependency side effects may already have changed storage, including storage reachable through aliases.

### Delegation

When both probe conditions pass, the function calls F-1c61b1704268 with five arguments, in this order:

1. The original primary reference.
2. The original requested displacement.
3. The integer value 1.
4. The original first optional output pointer.
5. The original second optional output pointer.

Null output pointers remain null when forwarded. A requested displacement of zero remains zero when forwarded, even though the probes used an effective displacement of 12.

The function returns the delegated result without a visible transformation. It does not restore the earlier output zeros, restore shared state, or perform another explicit status read after delegation.

## Side effects

### Direct memory accesses

The visible body directly performs:

- Up to two four-byte zero stores to optional outputs, in argument order.
- One read of the default-status shared location.
- For a nonnull primary reference, two separate reads through that reference.
- For a nonnull primary reference whose probe calls both return, one read of the probe-rejection-status shared location.

The function does not visibly allocate, release, copy, or retain a buffer. It performs no explicit ownership transfer. These observations do not establish the memory behavior of either dependency.

### Aliasing and access order

Output initialization may alter subsequent inputs if the output storage aliases the integer read through the primary reference. The primary-reference reads observe memory after the initial zero stores, not necessarily the value present on entry.

If both outputs identify the same four-byte location, that location receives two zero stores in the specified order. Partially overlapping output regions likewise undergo ordered four-byte writes; portability, alignment, and validity of such accesses are not established.

If an output aliases shared status storage, initialization may affect the status value subsequently read. The shared values must not be treated as immutable constants merely because they are used as return statuses.

### Dependency effects

Both dependencies may have effects not described by the supplied evidence. In particular, the contract does not assume that they:

- Preserve shared state;
- Preserve the integer reachable through the primary reference;
- Leave the output storage unchanged;
- Avoid allocation or deallocation;
- Return normally.

The first probe may affect the second probe’s argument. Either probe may affect the probe-rejection status read that follows. The body contains no visible recovery or rollback mechanism if a call fails to return normally.

## Dependencies

| Opaque function ID | Established interaction | Unknown behavior |
|---|---|---|
| F-c19b4c0cb7f7 | Called exactly twice on a normally completing nonnull-primary path. Each call receives one integer-derived argument. A byte from the first result and bits 8–15 of the second result control delegation. | Argument meaning, accessed memory, result semantics, error signaling, ownership effects, and other side effects. |
| F-1c61b1704268 | Called once only when both probe conditions pass. Receives the primary reference, original displacement, constant 1, and both original output pointers. Its result becomes this function’s result. | Meaning of the constant, output formats, success/error conventions, memory ownership, and state changes. |

The two shared status locations are additional state dependencies. Their values are read at the points described above; their initialization and meaning are unknown.

## Acceptance examples

These examples are conditional specifications, not reports of executed tests. Each assumes valid memory accesses and normally returning dependencies unless stated otherwise.

### Null primary reference

Given a null primary reference and two valid output slots:

- The first slot receives zero, then the second receives zero.
- The default status is read after those stores.
- Neither dependency is called.
- The returned value is that default status.

If either output pointer is null, its store is omitted without suppressing the other store.

### Zero displacement affects probes, not delegation

Given a nonnull primary reference holding 4096, requested displacement zero, and no intervening change to that stored value:

- Each probe receives 4122: 4096 plus 12 plus 14.
- If the probe predicate passes, delegation receives requested displacement zero, not 12.

### Nonzero displacement

Given a stored value of 4096 and requested displacement 4, with no intervening mutation or arithmetic overflow:

- Both probe arguments are 4114.
- Any delegation receives requested displacement 4.

Negative nonzero displacements are not visibly rejected. Their acceptability also depends on arithmetic validity and dependency requirements.

### Nonzero first byte still allows the second call

Given a first probe result whose retained byte is 1:

- The second probe still occurs.
- The probe-rejection status is then read.
- Delegation does not occur, regardless of the second result.
- The captured probe-rejection status is returned.

### Nonzero complete results can pass

Given a first result of hexadecimal 00000100 and a second result of hexadecimal 00000001:

- The first result’s retained byte is zero.
- Bits 8–15 of the second result are zero.
- Delegation occurs despite both complete results being nonzero.

Conversely, a zero first result and a second result of hexadecimal 00000100 reject delegation.

### Mutation between probes

Given an initial stored value of 4096 and requested displacement 4, suppose the first probe changes the value reachable through the primary reference to 8192:

- The first probe receives 4114.
- The second probe receives 8210, provided that storage remains valid.
- The function does not reuse the first computed probe argument.

### Output aliasing the primary storage

Given that the first optional output points to the same valid four-byte storage read through the primary reference:

- Initialization writes zero to that storage before either probe argument is constructed.
- With requested displacement zero and no later mutation, the first probe receives 26.
- The entry-time stored value does not control that argument.

### Delegated output and return values

If the gate passes and F-1c61b1704268 writes either output and returns a particular 32-bit result:

- Those writes are not explicitly undone by F-dc4dce183bd3.
- That result is returned rather than either captured shared status.

## Uncertainties

The following remain **documented_only**:

- Exact ABI, pointer width, alignment requirements, and integer-overflow behavior.
- Whether address-like additions wrap at a machine boundary or encounter another execution constraint. Examples above avoid overflow.
- The meaning and signedness of the stored primary integer beyond its visible arithmetic use.
- The storage layout behind the computed probe arguments.
- Whether F-c19b4c0cb7f7 reads memory, computes a value, or performs another operation.
- The meanings of the output slots, status values, and delegated constant argument.
- Dependency behavior on invalid inputs, including faults, exceptions, nonlocal exits, and nontermination.
- Ownership, retention, allocation, and release behavior inside dependencies.
- Thread-safety and synchronization requirements for shared state.

The description establishes two visible probe invocations, not volatile or atomic semantics for the underlying accesses. It provides no concurrency guarantees.

## Evidence

**Established static evidence:** The supplied function representation shows ordered optional output initialization, an initial shared-status read, a null-primary branch, zero-displacement substitution for probe construction, two sequential probe calls with separate primary-storage reads, a later shared-status read, a two-part gate, and conditional delegation with the original arguments plus constant 1.

**Inferred intent:** The arrangement resembles a precondition or format gate before a more substantial operation. That interpretation does not define acceptance semantics beyond the explicit byte and bit tests.

**Unknown:** Dependency implementations, shared-state definitions, broader caller contracts, and runtime behavior are unavailable. No runtime tests have been performed.
