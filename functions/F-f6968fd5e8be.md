<!-- Trusted provenance: F-f6968fd5e8be; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-f6968fd5e8be

## Purpose

F-f6968fd5e8be constructs a contiguous sequence of 16-bit elements from a context-dependent input stream. Control values select runs of literal values from either of two reader dependencies, or runs of zeros.

The requested element count is also passed to a preparation dependency. Successful completion requires the requested output length to coincide with the end of a run. If the output limit is reached partway through a run, the function invokes a cleanup dependency and returns zero, even though it has already written the requested number of elements.

**Established static behavior:** the function returns the prepared destination address only after successful preparation and completion at a run boundary, or immediately after successful preparation when the requested count is nonpositive.

**Inferred intent:** the function appears to decode a compact run-based representation into 16-bit output storage. The roles of the preparation and cleanup dependencies resemble allocation and release, but their actual ownership contracts are not established.

## Interface

The function has two inputs and one 32-bit return value in the supplied representation.

| Item | Representation | Role |
|---|---|---|
| Context | 32-bit address-valued input | Identifies an object whose 32-bit field at byte offset 28 is read before preparation. The original context is also supplied to both reader dependencies. |
| Requested count | Signed 32-bit integer | Requested number of 16-bit output elements. Passed unchanged to the preparation dependency. |
| Return | 32-bit value used as an address, or zero | Prepared destination address on the successful path; zero after a reported preparation failure or incomplete terminal run. |

The function uses a writable 32-bit preparation-status object, initially zero. It supplies that object's address to the preparation dependency and examines its value afterward.

### Context and destination layout

Only one context field is directly accessed by this function:

| Byte offset | Width | Use |
|---|---:|---|
| 28 | 32 bits | Captured handle supplied to the preparation dependency and, if needed, the cleanup dependency. |

No other context fields are directly interpreted here. Reader dependencies may access or change additional context state; their behavior is unknown.

The destination consists of consecutive 16-bit elements. Element zero is stored at the returned base address, element one two bytes later, and so on. Each literal is reduced to its low 16 bits before storage. Zero runs store a 16-bit zero.

The supplied evidence does not establish byte order, alignment guarantees, or whether unaligned 16-bit stores are supported.

### Preconditions

Defined execution requires:

- The context field at byte offset 28 to be readable, even for zero or negative requested counts.
- Each invoked dependency to accept the arguments supplied to it.
- Whenever output stores occur, the prepared destination to support those stores.
- Address arithmetic and accesses to remain valid in the target execution environment.

The function does not validate these conditions. It does not directly check the destination for zero, check the available input length, or validate reader results as errors.

## Behavior

### Preparation

The function first reads and captures the context's 32-bit field at byte offset 28. It initializes its preparation-status object to zero, then calls F-6dd003795db7 with six ordered arguments:

1. The captured handle.
2. The integer value 2.
3. The integer value 0.
4. The requested count, unchanged.
5. The integer value 0.
6. The address of the writable preparation-status object.

The return value is retained as the prospective destination address.

After the call:

- If the status object is nonzero, the function returns zero. It performs no reader calls, output stores, or explicit cleanup call.
- If the status object remains zero, processing proceeds using the returned destination without a separate validity check.

The status object starts at zero, so a preparation dependency that leaves it untouched is treated as reporting success.

### Nonpositive requested counts

After successful preparation, the function compares an output position initially equal to zero against the signed requested count.

For a requested count of zero or less, it immediately returns the preparation result. It does not read a control value, write output, or invoke cleanup.

Preparation is therefore still observable for nonpositive counts. A negative count is neither rejected locally nor converted to an unsigned length before being passed to F-6dd003795db7.

### Control values and run lengths

For a positive requested count, the function obtains each control value by calling F-0b247502190b with the original context.

The control value is interpreted as an unsigned 32-bit value. Its relevant fields are:

| Bits | Meaning |
|---|---|
| 0–5 | Encoded run length minus one |
| 6 | Selects the literal reader when bit 7 is clear |
| 7 | Selects a zero run |
| 8–31 | Do not affect this function's run selection or length |

Every run has a declared length from 1 through 64. The length is one greater than the value in bits 0–5.

Run selection has this precedence:

| Bit 7 | Bit 6 | Run behavior |
|---:|---:|---|
| 0 | 0 | Read each literal through F-0b247502190b |
| 0 | 1 | Read each literal through F-7efb5338f0fa |
| 1 | Either | Write zeros without reading literal payloads |

Bit 6 has no effect when bit 7 is set.

A control value is used as returned. The function does not first narrow it to eight bits, although bits above bit 7 do not affect the decisions shown here.

### Literal runs

For each literal element allowed by both the declared run length and the remaining output capacity, the function:

1. Calls the selected reader with the original context.
2. Retains the low 16 bits of the returned value.
3. Writes those 16 bits to the next destination element.
4. Advances the output position.

F-0b247502190b therefore serves two roles: reading control values and reading literal values for one run mode.

The function applies no independent error or range check to either reader's result. Returned bits are interpreted as control data or narrowed literal data according to the current role.

A literal run consumes only the payload values actually needed before the output limit is reached. If the output fills before the declared run ends, the remaining payload values are not read by this function.

### Zero runs

For each zero element allowed by both the declared run length and the remaining output capacity, the function writes a 16-bit zero and advances the output position.

A zero run makes no literal-reader calls. Its control value has already been obtained through F-0b247502190b.

### Run completion and terminal outcomes

The function distinguishes completion of a run from merely filling the output.

**Complete run with output capacity remaining:** another control value is read and processing continues.

**Complete run that exactly fills the output:** the function returns the prepared destination address. No additional control value is read, and cleanup is not invoked.

**Output filled before the declared run ends:** the function stops processing the run, invokes F-bf7242c98d06 with the captured handle and the prepared destination address, then returns zero.

Consequently, a positive request succeeds only when the total lengths of fully processed runs equal the requested count. A final run that exceeds the remaining capacity causes failure rather than accepted truncation.

There is no rollback of stores or reader activity before cleanup. On this failure path, the requested number of elements has already been written, and input associated with the processed portion has already been read.

### Integer and address behavior

The requested count and output-position comparisons use signed 32-bit quantities. Under ordinary valid execution with a positive signed count, the output position advances from zero up to that count.

The decoded run length is always between 1 and 64, so the per-run counter does not approach signed overflow.

Destination addresses use a two-byte stride. Although the logical output position is bounded by the requested count, that does not establish that address calculations or the resulting memory accesses are valid. The behavior of very large allocations, address wrapping, and invalid stores is not established.

## Side effects

The directly established ordering is:

1. Read the context field at byte offset 28.
2. Call F-6dd003795db7.
3. Inspect the preparation status.
4. If appropriate, alternate control reads with literal reads and destination stores.
5. On an incomplete terminal run, call F-bf7242c98d06 after the partial run's reads and stores.
6. Return the selected result.

The function does not directly write the context field. It also does not reread that field before cleanup: cleanup receives the handle captured before preparation. A dependency may change the context or its field, but this does not change the saved handle used here.

The function does not directly access global state. This is not evidence that its dependencies preserve globals, avoid external I/O, or leave other objects unchanged.

### Ownership and lifetime

The observable lifecycle is:

- Preparation supplies the destination value.
- Successful processing returns that value without explicit cleanup.
- An incomplete terminal run passes that value to cleanup.
- A nonzero preparation status causes return without explicit cleanup, regardless of the preparation return value.

These observations do not establish allocation ownership, the caller's release obligations, or whether cleanup frees memory. No such guarantee should be inferred solely from the call pattern.

## Dependencies

| Opaque ID | Invocation and established use | Unknown contract details |
|---|---|---|
| F-6dd003795db7 | Called once with the captured handle, 2, 0, requested count, 0, and a writable 32-bit status address. Its return is treated as the destination address. | Allocation policy, meaning of constants, status values, destination validity, ownership, and other side effects. |
| F-0b247502190b | Called with the original context for every control value and for each literal in the bit-7-clear, bit-6-clear mode. | Input-unit width, cursor behavior, end-of-input behavior, error signaling, and state changes. |
| F-7efb5338f0fa | Called with the original context for each literal in the bit-7-clear, bit-6-set mode. | Input encoding, cursor behavior, error signaling, and state changes. |
| F-bf7242c98d06 | Called with the previously captured handle and destination after the requested output limit interrupts a run. Its return, if any, does not affect the result. | Whether storage is released, invalidated, retained, or otherwise changed; additional side effects. |

These dependencies are not assumed to preserve globals, preserve the context, or acquire or release memory.

## Acceptance examples

These examples are conditional static expectations, not executed tests. Unless otherwise specified, preparation reports zero status, returns usable storage, and all dependencies return normally.

1. **One literal through the shared reader.**  
   Requested count: 1. Control value: hexadecimal 00. The function obtains one literal through F-0b247502190b, stores its low 16 bits at destination element zero, and returns the destination. It reads no further control.

2. **Two literals through the alternate reader.**  
   Requested count: 2. Control value: hexadecimal 41. The function calls F-7efb5338f0fa twice, storing each result's low 16 bits in order. The complete two-element run exactly fills the output, so the destination is returned.

3. **Literal narrowing.**  
   A literal reader returns hexadecimal 12345678 during a literal run. The corresponding stored element is hexadecimal 5678. The function does not reject the value because its upper bits are nonzero.

4. **Zero-run precedence.**  
   Requested count: 3. Control value: hexadecimal C2. Both mode bits are set, but bit 7 takes precedence. Three zero elements are stored, no literal reader is called, and the destination is returned.

5. **Multiple complete runs.**  
   Requested count: 3. A hexadecimal 01 control produces two literals through F-0b247502190b. The next control is hexadecimal 80 and produces one zero. The destination contains the two narrowed literals followed by zero, and is returned.

6. **Final literal run exceeds the remaining capacity.**  
   Requested count: 2. Control value: hexadecimal 02 declares three literals. The function reads and stores two literals, does not read the third, calls F-bf7242c98d06, and returns zero.

7. **Final zero run exceeds the remaining capacity.**  
   Requested count: 1. Control value: hexadecimal 81 declares two zeros. The function writes one zero, invokes cleanup, and returns zero.

8. **Upper control bits do not change decoding.**  
   Requested count: 1. Control value: hexadecimal 00000180. The low mode and length fields select a one-element zero run. The function stores zero and returns the destination.

9. **Nonzero preparation status.**  
   Preparation changes the status object to any nonzero 32-bit value. The function returns zero without reading control values, storing destination elements, or explicitly invoking cleanup.

10. **Zero or negative requested count.**  
    Preparation receives the count unchanged. If status is zero, the function returns the preparation result without decoding. If status is nonzero, it returns zero instead.

## Uncertainties

The following remain **documented_only**:

- The concrete context type and the meaning of its offset-28 field.
- Whether the preparation dependency allocates memory and whether its argument value 2 denotes element width.
- Reader encodings, input bounds, and failure conventions.
- Destination ownership, release obligations, and cleanup semantics.
- Endianness, alignment requirements, and invalid-access behavior.
- Dependency mutations of globals, context state, or external resources.
- Whether callers deliberately rely on nonpositive counts.
- Whether exact run-boundary completion is an intended format rule or an implementation consequence.

The zero return value does not uniquely identify a cause. It covers explicit failure paths and can also be the preparation result returned on a successful nonpositive-count path.

## Evidence

This contract is based solely on static examination of the supplied function body and dependency mapping. Established behavior includes the direct context-field read, dependency argument order, status gating, control-bit interpretation, 16-bit stores, signed count comparisons, exact-run-boundary requirement, and cleanup-call conditions.

Descriptions of decoding intent and allocation-like lifecycle are explicitly inferential. Dependency internals and runtime environmental guarantees are not available.

No runtime tests have been performed.
