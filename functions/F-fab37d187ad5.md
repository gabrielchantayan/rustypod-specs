<!-- Trusted provenance: F-fab37d187ad5; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-fab37d187ad5

## Purpose

F-fab37d187ad5 dispatches an operation using a request record, a referenced state record, and an opaque auxiliary value. It first captures the referenced state location, invokes a preliminary dependency, and copies one request word into that captured state. It then selects either a conditional three-stage dependency chain or a status derived from one state byte. A final dependency receives the request before the selected result is returned.

The established behavior is the ordering of memory accesses, dependency calls, branch selection, and result propagation. The meanings of the request fields, state fields, auxiliary value, and returned statuses are not established. Descriptions such as “request” and “state” are structural labels, not claims about a particular application protocol.

## Interface

### Inputs

The function receives two arguments:

- **Request address:** an address supporting the request-record accesses described below.
- **Auxiliary value:** an opaque 32-bit value forwarded unchanged to whichever operational dependencies execute. Its interpretation as an integer, address, handle, flags, or another representation is unknown.

The request layout visible to this function is:

| Byte offset | Width | Observed use |
|---|---:|---|
| 0 | 4 bytes | A word read after the preliminary dependency returns and copied into the captured state record. |
| 4 | 4 bytes | A state-location representation read before the preliminary dependency is invoked. |
| 8 | 1 byte | A selector read after the state-word write; equality to 2 selects the dependency chain. |

Only one byte at offset 8 is inspected. Adjacent bytes do not participate in that comparison.

The referenced state layout visible to this function is:

| Byte offset | Width | Observed use |
|---|---:|---|
| 4 | 4 bytes | Destination of the copied request word. |
| 17 | 1 byte | Status-selection byte, read only when the request selector is not 2. |

These accesses do not establish either record’s total size. Dependencies may access additional memory under contracts not supplied here.

### Output

The function returns an integer result:

- On the selector-2 path, the result comes from the first operational dependency that returns nonzero, or from the final operational dependency if both preceding results are zero.
- On the other path, the result is exactly 1 or 8, according to the state byte at offset 17.

The supplied representation explicitly identifies the request words and auxiliary value as 32-bit quantities. It represents the result and dependency results as integer scalars, consistent with a 32-bit target, but does not independently establish the complete ABI or signedness conventions. No result is narrowed, masked, or otherwise transformed by this function.

### Preconditions

For an execution that returns normally:

- The initial 4-byte request field at offset 4 must be readable.
- After the preliminary dependency returns, the request word at offset 0 must be readable.
- The captured state location plus offset 4 must support a 4-byte write.
- After that write, the request selector byte at offset 8 must be readable.
- If the selector is not 2, the captured state location plus offset 17 must be readable.
- Every invoked dependency’s own preconditions must hold.

There is no visible null check, length check, address-range check, or recovery path for invalid memory. Alignment requirements, address-overflow behavior, and handling of memory faults are not established.

## Behavior

### Capture before the preliminary dependency

The function first reads the 4-byte state-location field from request offset 4 and retains that value for its subsequent direct state accesses and state-directed dependency calls.

It then calls F-0b4658e20c99 with no explicit arguments. No return value from this dependency is used.

The capture occurs before this call. Consequently, changing the request’s state-location field during F-0b4658e20c99 does not redirect this function’s subsequent direct state accesses. The dependency’s other effects remain relevant: it may change memory that the function has not yet read.

### Copy after the preliminary dependency

After F-0b4658e20c99 returns normally, the function reads the 4-byte word at request offset 0 and writes it to offset 4 of the previously captured state location.

The copied value is therefore the value observed after the preliminary dependency, not necessarily the value present on entry.

The function next reads the single selector byte at request offset 8. This ordering is material if memory overlaps: the 4-byte state write can affect the subsequently read selector when their storage overlaps. The function does not visibly reject overlap or take a snapshot of the entire request.

### Selector equal to 2

If the selector byte equals 2, the function executes the following conditional dependency chain:

1. F-fa3b853f090d receives the request address, the auxiliary value, and a zero-valued third argument.
2. If its result is nonzero, that result is selected and the remaining operational dependencies are skipped.
3. If its result is zero, F-533f1c9e9e0b receives the captured state location, the auxiliary value, and a zero-valued third argument.
4. If that result is nonzero, it is selected and the final operational dependency is skipped.
5. If both preceding results are zero, F-0398acce38ad receives the captured state location, the auxiliary value, and two zero-valued arguments. Its result is selected without another zero/nonzero decision.

The zero constants are established argument values. Their semantic meanings and exact formal parameter types are unknown.

This chain preserves the distinction between the request address passed to its first dependency and the captured state location passed to its second and third dependencies. In particular, the first dependency may observe a request whose state-location field has changed since entry, while the later dependencies still receive the earlier captured location.

The selector is not reread during this chain. Mutations made by an operational dependency do not cause the function to switch to the alternative branch or restart dispatch.

### Selector not equal to 2

For every other selector byte value, none of the three operational dependencies is invoked.

Instead, the function reads the single byte at offset 17 of the captured state location:

- A zero byte selects result 1.
- Any nonzero byte selects result 8.

No additional interpretation of this byte is visible. In particular, distinct nonzero byte values have identical result-selection behavior.

This byte is read after the state-word write. Any valid aliasing effects of that earlier write are therefore reflected in the byte observed here.

### Final call and return

After either branch selects a result, the function calls F-96dd78c4ca33 with the request address.

No return value from this final dependency is used. If it returns normally, the function returns the result already selected.

There is no visible early return that bypasses this final call after ordinary result selection. However, the supplied body establishes no exception handling, nonlocal-exit handling, or fault recovery. It does not establish that the final call occurs if an earlier dependency fails to return normally or a memory access faults.

## Side effects

### Direct memory effects

The function visibly performs one direct memory write: a 4-byte copy from request offset 0 to offset 4 of the captured state location.

That write occurs before branch selection and is not conditional on an operational dependency succeeding. No rollback of it is visible.

The ordered direct accesses are:

1. Read the state-location word from request offset 4.
2. Invoke F-0b4658e20c99.
3. Read the request word at offset 0.
4. Write that word at captured state offset 4.
5. Read the request selector at offset 8.
6. If the selector is not 2, read the byte at captured state offset 17.

Dependency effects are interleaved with these accesses according to the behavior above. This ordering describes the supplied function’s access sequence; it is not a claim about machine-level atomicity, hardware transactions, or synchronization.

### Ownership and lifetime

No allocation, deallocation, reference-count operation, or ownership transfer is directly identifiable in this function.

The final dependency’s position does not establish that it releases or destroys the request. Likewise, the preliminary dependency is not established to acquire a lock, initialize state, or preserve any globals.

The supplied evidence does not guarantee that the request or state remains usable by the caller after return. That guarantee requires the dependency contracts. Conversely, this function does not visibly access either record after F-96dd78c4ca33 returns.

### Errors and state changes

The selector-2 chain treats zero as the condition for proceeding to the next dependency and nonzero as the condition for stopping early. This does not establish whether zero means success, failure, absence, or “not handled.”

The function does not normalize dependency results into the alternative branch’s values 1 and 8. It also does not visibly translate errors, retry operations, emit diagnostics, or restore prior state.

No direct global-variable access is shown. This is not evidence that invoked dependencies preserve global state.

## Dependencies

| Opaque dependency | Invocation condition | Explicit arguments | Result use |
|---|---|---|---|
| F-0b4658e20c99 | After capturing the state location, before copying the request word | None | No result consumed |
| F-fa3b853f090d | Selector equals 2 | Request address; auxiliary value; zero | Nonzero stops the chain; zero continues |
| F-533f1c9e9e0b | Selector equals 2 and the preceding result is zero | Captured state location; auxiliary value; zero | Nonzero stops the chain; zero continues |
| F-0398acce38ad | Selector equals 2 and both preceding results are zero | Captured state location; auxiliary value; zero; zero | Selected directly |
| F-96dd78c4ca33 | After result selection on every normally progressing branch | Request address | No result consumed |

Unknown dependency effects must not be replaced with assumptions about memory preservation, ownership, synchronization, or global-state stability.

## Acceptance examples

These are conditional acceptance criteria derived from static evidence, not reports of executed tests. Each assumes valid accesses and normal returns from all invoked dependencies.

### Preliminary mutation affects the copied value

Suppose F-0b4658e20c99 changes the request word at offset 0 from 10 to 20. The write to captured state offset 4 must use 20.

A contract requiring the entry-time value 10 would contradict the observed read order.

### Preliminary mutation does not redirect the captured state

Suppose the initial request identifies state A, and F-0b4658e20c99 changes that field to identify state B.

The direct state-word write must still target state A. The alternative branch’s state-byte read, if taken, must also use state A. On the selector-2 path, the second and third operational dependencies, when reached, must receive state A.

This does not constrain which state the first operational dependency accesses internally through the request.

### First operational result stops the chain

Suppose the selector observed after the state write is 2 and F-fa3b853f090d returns 5.

F-533f1c9e9e0b and F-0398acce38ad must not be called. F-96dd78c4ca33 must be called next, and normal completion returns 5.

### Second operational result stops the chain

Suppose the selector is 2, the first operational dependency returns zero, and F-533f1c9e9e0b returns 8.

F-0398acce38ad must not be called. After the final dependency returns normally, the function returns 8. This result follows from dependency propagation, not from reading the state byte at offset 17.

### Final operational result is retained, including zero

Suppose the selector is 2 and both preceding operational dependencies return zero.

F-0398acce38ad must be called. If it returns zero, normal completion returns zero; if it returns another integer value, normal completion returns that value.

### Alternative branch distinguishes only zero from nonzero

Suppose the selector is 3. If the state byte at offset 17 is zero when read, the selected result is 1. If it is 255, the selected result is 8.

Neither case invokes any operational dependency. Both invoke the preliminary and final dependencies.

### Overlap is evaluated in access order

Suppose a valid storage arrangement makes the state-word destination overlap the request selector. If the write leaves the subsequently read selector equal to 2, the selector-2 chain is taken, regardless of the selector’s entry-time value.

This criterion applies only where the target’s memory and aliasing rules permit those accesses.

## Uncertainties

The following are **documented_only**:

- The semantic purpose and complete layout of both records.
- The complete target ABI, address representation, integer signedness, alignment rules, and byte order.
- Whether overlapping records are valid supported inputs.
- The operational meanings of zero, nonzero dependency results, and fixed results 1 and 8.
- The interpretation and validity constraints of the auxiliary value.
- Dependency memory effects, ownership effects, global effects, synchronization behavior, and failure modes.
- Behavior under concurrent mutation, invalid addresses, address arithmetic overflow, faults, exceptions, or nonlocal exits.
- Record lifetimes and caller obligations after the final dependency.

**Inferred intent:** the three-stage chain resembles conditional fallback because later stages run only after zero results. That interpretation is not needed to apply the established contract and does not assign success or failure semantics.

## Evidence

The supplied function body establishes the field widths explicitly represented as 4-byte words, byte-access offsets, capture-before-call ordering, copy-before-selector ordering, conditional call sequence, literal zero arguments, fixed alternative results, and final dependency invocation before return.

Only this body was available for analysis. Dependency implementations, type definitions, ABI documentation, and independent runtime observations were not supplied.

No runtime tests have been performed. The acceptance examples specify conditional expected behavior; they are not execution evidence.
