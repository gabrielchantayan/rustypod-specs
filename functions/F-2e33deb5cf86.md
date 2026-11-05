<!-- Trusted provenance: F-2e33deb5cf86; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-2e33deb5cf86

## Purpose

F-2e33deb5cf86 records a requested byte count in a context, programs a memory-mapped processing interface for part of an input range, and delegates the remaining sub-block portion to F-b490bea51f76.

The established behavior divides work using a 64-byte boundary and a repeated decrement of 65,472 bytes. The repeated-transfer decision depends on a separately loaded configuration word whose value is not supplied. Consequently, the evidence does **not** establish that every invocation partitions its input into valid, nonoverlapping transfers totaling the requested size.

The register layout and busy-bit polling suggest a hardware-assisted block-processing operation. The particular algorithm, the meaning of the command values, and the intended relationship between the configuration word and 65,472 are inferred or unknown, not established.

No runtime tests have been performed.

## Interface

### Inputs

The function accepts three arguments:

| Argument | Established interpretation |
|---|---|
| Context address | Address used to access three 32-bit words at byte offsets 0x40, 0x44, and 0x48. The same address is passed to F-b490bea51f76. |
| Input address | Address value written to the processing interface and used to derive the address passed to F-b490bea51f76. |
| Requested count | Unsigned 32-bit quantity added to the context count and used to control processing. |

The supplied representation uses 32-bit integer address operands. Address advancement is therefore described as 32-bit address arithmetic, not as an assurance that the resulting address remains within a valid allocation. Valid addressing across a wrap is not established.

There is no returned result or status value.

### Context layout

Only these context fields are directly accessed:

| Byte offset | Width | Direct use |
|---|---:|---|
| 0x40 | 32 bits | Low count word, read and replaced with its sum with the requested count. |
| 0x44 | 32 bits | High count word, read and incremented only if the low-word addition carries. |
| 0x48 | 32 bits | Command word, read for transfer initiation and replaced with decimal 10 at specified points. |

The first two fields behave as a split 64-bit accumulator. No other context layout is established.

### External locations

The function directly accesses four 32-bit memory-mapped words:

| Address | Direct use |
|---|---|
| 0x38000000 | Receives command values; bit 0 is polled as a busy indication. |
| 0x38000080 | Receives decimal 1 once per invocation that reaches interface setup. |
| 0x38000084 | Receives an input address for each initiated transfer. |
| 0x3800008c | Receives an initial configuration-derived value, then possibly an aligned remainder length. |

A separate 32-bit configuration word supplies the repeated-transfer threshold and the initial value written to 0x3800008c. Its value is unknown.

### Preconditions and ownership

For defined operation of the described accesses:

- The context fields must be accessible for their respective reads and writes.
- The memory-mapped words must support the accesses and polling performed.
- Input address values must be meaningful to the processing interface and to F-b490bea51f76 for the portions each consumes.
- The configuration and device behavior must permit progress if termination is required.

The function does not directly allocate, release, or transfer ownership of memory. F-b490bea51f76’s ownership behavior is unknown. The evidence supplies no non-aliasing, synchronization, or reentrancy guarantee.

## Behavior

### 1. Record the requested count

The low count word is read, added to the original requested count, and written back as a 32-bit result.

Carry is detected by comparing that result, as an unsigned value, with the requested count. If the result is smaller, the high count word is read, incremented, and written back. Otherwise, the high word is not directly accessed.

Under ordinary 32-bit word arithmetic, this is addition of the requested count to a split 64-bit counter, wrapping at 64 bits. The high-word expression is presented as signed in the supplied representation; signed overflow semantics are not independently established. Wrapping of the high word is the natural machine-word interpretation, rather than a demonstrated language-level guarantee.

The count records the **requested** quantity before any interface setup, transfer completion, or dependency call. There is no rollback on nontermination or failure.

### 2. Initialize the interface

After the counter update:

1. The configuration word is read and retained as the unsigned repeated-transfer threshold.
2. Decimal 1 is written to 0x38000080.
3. The configuration word is read again and that value is written to 0x3800008c.

The two configuration reads are distinct in the supplied representation. Equality of their values requires stability of the configuration word; it must not be assumed merely because both accesses name the same storage.

The threshold retained from the first read is used for every repeated-transfer comparison. It is not refreshed inside that loop.

### 3. Perform repeated fixed-step submissions

The function maintains a current input address and an unsigned 32-bit remaining count, initially equal to the input address and requested count.

While the retained threshold is strictly less than the remaining count, each iteration performs the following observable sequence:

1. Read the context command word at offset 0x48 and write it to 0x38000000.
2. Subtract 65,472, or 0xffc0, from the remaining count.
3. Form the next input address by adding 65,472 to the current address.
4. Write the current, unadvanced input address to 0x38000084.
5. Write decimal 10 to the context command word.
6. Poll bit 0 of 0x38000000 until it is clear.
7. Continue using the advanced input address.

The command-register write precedes the source-address write. The context command becomes 10 before completion polling finishes.

No length-register write occurs inside this loop. Repeated submissions therefore rely on the earlier value written to 0x3800008c and on device behavior.

The subtraction is not guarded by a direct comparison against 65,472. The actual guard is the unknown threshold comparison. If that threshold permits entry with fewer than 65,472 bytes remaining, unsigned subtraction wraps. The evidence contains no validation or alternate handling for this case.

There is no timeout or iteration limit. Progress depends both on busy-bit clearance and on the arithmetic eventually making the remaining count no greater than the retained threshold.

### 4. Optionally submit the aligned remainder

Once the repeated-transfer loop exits, the function divides the remaining count into:

- An aligned prefix obtained by clearing its lowest six bits.
- A tail obtained by retaining only its lowest six bits.

The aligned prefix is a multiple of 64. The tail is between 0 and 63 inclusive.

If the remaining count exceeds 63, the function:

1. Writes the aligned-prefix length to 0x3800008c.
2. Reads the context command word and writes it to 0x38000000.
3. Writes the current input address to 0x38000084.

It does not wait at this point and does not yet replace the context command with 10.

If the remaining count is 63 or less, none of those three accesses occurs.

### 5. Invoke the tail dependency

F-b490bea51f76 is called with:

1. The original context address.
2. The current input address plus the aligned-prefix length.
3. The tail length, from 0 through 63.

This call occurs even when the tail length is zero, provided execution reaches this stage. In particular, an original request of zero does not skip it.

When an aligned remainder was submitted, the dependency is invoked **before** the subsequent completion wait. Thus, the evidence permits the dependency to run while that transfer remains busy. It does not establish whether this overlap is necessary, harmless, or internally synchronized.

The call’s return value, if any, is not used. Its internal effects, memory ownership behavior, and treatment of a zero-length input are unknown.

### 6. Complete the optional aligned remainder

If the remaining count exceeds 63 and F-b490bea51f76 returns:

1. Bit 0 of 0x38000000 is polled until clear.
2. Decimal 10 is written to the context command word.
3. The function returns.

If the remaining count is at most 63, the function returns after the dependency without this final poll or command-field write.

The final poll reads the live interface state after the dependency returns. It must not be described as necessarily observing an unchanged command or the same outstanding operation: the dependency’s effects on the interface are unknown.

## Side effects

### Direct state changes

The function directly changes:

- The context low count word on every invocation reaching that write.
- The context high count word only upon low-word carry.
- The context command word during each repeated submission and after completion of an optional aligned remainder.
- The four memory-mapped words according to the ordering above.

The input payload is not directly read or written by ordinary data accesses in this function. Its address is submitted to the interface and passed to the dependency. Whether either consumer reads, writes, retains, or otherwise uses that memory is outside the established evidence.

### Ordering and visibility

The source-level ordering establishes several important distinctions:

- Count accumulation precedes all interface programming.
- Each repeated transfer is polled before another repeated transfer is submitted.
- During repeated processing, the context command is set to 10 before polling.
- For the optional aligned remainder, that context update occurs only after the dependency and polling.
- The dependency always precedes the final poll when an aligned remainder exists.

These are ordering statements from the supplied representation, not proof of hardware memory barriers, bus completion rules, or cross-thread visibility.

No direct state restoration occurs. The interface setup and last written register values remain subject to device behavior and dependency side effects.

### Errors and nontermination

There is no explicit error return, input validation, cancellation, retry bound, or timeout.

A permanently set busy bit prevents progress at the relevant polling point. Invalid memory or device accesses are not caught here. If the dependency does not return, neither the final polling nor the subsequent context update occurs.

The count may therefore reflect a request that never completes.

## Dependencies

### F-b490bea51f76

**Established call contract:** receives the context address, an address immediately following the aligned prefix of the current remainder, and an unsigned tail length no greater than 63.

It is called once if execution reaches the tail-processing stage. A zero tail does not suppress the call.

**Unknown behavior:** reads and writes, global changes, register changes, allocation, release, pointer retention, error handling, and zero-length semantics. No preservation guarantee may be inferred.

### Configuration word

The first read fixes the loop threshold for the invocation. The second supplies the initial interface length/configuration value.

Its actual value, mutability, and intended units are undocumented. A value of 65,472 would make the repeated decrement and threshold naturally agree, but that value is not established.

### Processing interface

The function treats bit 0 at 0x38000000 as requiring a wait while set. It writes command, source-address, and length-like values to the listed registers.

The interface’s actual triggering rules, accepted lengths, access ordering requirements, and completion semantics remain unknown.

## Acceptance examples

These examples are conditional static expectations, not observed executions. Unless otherwise stated, configuration reads are stable, memory accesses succeed, address arithmetic does not wrap, and the dependency returns without altering the described context or interface state.

### Zero request

For a requested count of zero:

- The low count word is rewritten unchanged; no carry increment occurs.
- Interface initialization still occurs.
- No repeated or aligned transfer is submitted.
- F-b490bea51f76 receives the original input address and length zero.
- No busy-bit poll is performed by this function.

### Sub-block request

For a requested count of 37 and a threshold of at least 37:

- The count increases by 37.
- No transfer is submitted.
- The dependency receives the original input address and length 37.
- The context command is not directly replaced with 10.

### Exactly one block

For a requested count of 64 and a threshold of at least 64:

- The aligned remainder length is 64.
- The context command is written to the command register before the input address is written.
- The dependency receives input address plus 64 and length zero.
- The function subsequently waits for bit 0 to clear and then writes 10 to the context command.

### One block plus a tail

For a requested count of 65 and a threshold of at least 65:

- The aligned remainder length is 64.
- The dependency receives input address plus 64 and length 1.
- The dependency precedes the final completion wait.

### Stable threshold of 65,472

If both configuration reads yield 65,472:

- A request of exactly 65,472 skips the repeated loop and uses one aligned-remainder submission of that length.
- A request of 65,473 performs one repeated submission, advances the current address by 65,472, and delegates a one-byte remainder without an additional aligned submission.
- A request of 131,008 performs two repeated submissions, then delegates a 64-byte remainder through one aligned submission and a zero-length dependency call.

These expectations depend on the stated threshold value; they do not establish its actual value.

### Low-word carry

For a low count word of 0xfffffff0 and a requested count of 32, the new low word is 0x00000010 and the high word is incremented once. This occurs before transfer setup.

### Threshold permits underflow

For a threshold of zero and requested count of one, the repeated loop is entered. Its first subtraction produces 0xffff0041 under unsigned 32-bit arithmetic, not a one-byte tail. A contract claiming direct delegation of that one byte would contradict the supplied behavior.

## Uncertainties

The following are **documented_only**:

- The configuration word’s value and stability.
- Whether the register interface is DMA, another accelerator, or another kind of device.
- The algorithm and meaning of command value 10.
- Whether command writes trigger work immediately or only in combination with later register writes.
- Hardware ordering, cache coherence, alignment requirements, and memory barriers.
- The permitted input-memory range and device access direction.
- F-b490bea51f76’s effects, ownership rules, failure modes, and synchronization.
- Concurrency safety and behavior under aliasing between context, input, configuration, and device storage.
- Exact runtime consequences of address overflow and high-count-word overflow.
- Whether the displayed polling accesses correspond to appropriately volatile machine accesses.

None of these unknowns licenses an assumed fallback, validation step, preservation guarantee, or successful completion.

## Evidence

**Established static evidence:** three 32-bit context fields; low-word carry detection; two configuration reads; ordered interface writes; unsigned threshold comparison; repeated subtraction and address advancement by 0xffc0; 64-byte alignment masks; unconditional tail-dependency invocation upon reaching that stage; bit-0 polling; and command-field updates to decimal 10.

**Inferred intent:** accumulation of a total byte count, bulk processing through a hardware-like interface, and delegation of a short tail.

**Verification status:** this contract is based solely on the supplied static representation. No implementation execution, runtime tests, device observations, or dependency inspection have been performed.
