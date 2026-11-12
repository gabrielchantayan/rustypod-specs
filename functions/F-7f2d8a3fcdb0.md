<!-- Trusted provenance: F-7f2d8a3fcdb0; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-7f2d8a3fcdb0

## Purpose

F-7f2d8a3fcdb0 performs an ordered sequence of indirect calls, address calculations, memory reads, and memory writes. Its directly visible effects include propagating a captured 32-bit word to several locations, installing scaled call results into other locations, performing two byte-table-directed stores, updating a global destination, and attempting a call to address zero.

No application-level purpose is established. The available evidence does not justify classifying this operation as allocation, serialization, device initialization, resource release, or a transaction.

The contract below describes the visible operation sequence, conditional on dependencies returning and required memory accesses succeeding. It does not assume that unknown dependencies preserve memory, globals, or machine state.

## Interface

### Explicit inputs

The displayed interface has three parameters:

| Input | Width | Established use |
|---|---:|---|
| First argument | 32 bits | No direct use in the visible body. |
| Second argument | 32 bits | No direct use in the visible body. |
| Third argument, called **input word** below | 32-bit unsigned | Used in one transformed call argument and one store address. |

The absence of direct uses of the first two arguments does not establish that they are irrelevant to the complete execution. Indirect calls have incompletely represented calling conventions and may observe inherited machine state.

### Implicit inputs

Execution also depends on:

- Incoming return-link state, from which the first indirect-call target is calculated.
- The results and effects of 64 indirect call sites.
- A global word interpreted as an address.
- Memory at dynamically calculated addresses.
- An implicit pointer available near the end of execution.
- The validity and behavior of a call through address zero.

The available evidence does not establish a complete externally callable ABI.

### Outputs

The displayed interface declares no ordinary return value. Visible outputs are memory writes, dependency effects, and a global assignment.

A normal return is only conditional: the final call through address zero must return before the visible return can be reached. There is no established success value, error value, or completion flag.

### Width and notation

For this contract:

- A **word** is 32 bits.
- A **byte** is 8 bits.
- A **halfword** is 16 bits.
- The **low byte** of a value is its least significant eight bits.
- A paired result is the low and high 32-bit portions of a displayed 64-bit result.

Unsigned right shifts are logical. Unsigned left shifts retain only the low 32 bits.

Several address calculations instead use results represented as signed 32-bit integers. Their overflow behavior is not established by the supplied representation. Where those calculations are involved, acceptance examples assume representable arithmetic and valid resulting addresses. A universal modulo-addressing rule is not asserted.

## Behavior

### 1. Capture a word and perform the first byte store

The function invokes F-000000000001, F-000000000002, F-000000000003, and F-000000000004, in that order. No explicit arguments are shown for these calls.

The first call target depends on incoming return-link state. The other three targets are derived from fixed code-associated values.

The result of F-000000000004 is multiplied by four. Call the resulting address-like value the **initial base**.

The function then:

1. Reads a 32-bit word at initial base plus `0x74`.
2. Retains that value as the **captured word**.
3. Writes its low byte at initial base plus `1`.
4. Calls F-000000000005 with the initial base.

The read precedes the byte store, and the byte store precedes the call. If the locations overlap, the captured word still denotes the value obtained before that store.

### 2. Propagate the captured word to a stride-16 location

The function invokes F-000000000006 through F-000000000009 in numerical order, with no explicit arguments shown.

It uses the result of F-000000000009 to form an address by multiplying the result by `0x10` and adding `0x14`. It writes the full captured word there.

It then calls F-000000000010 with:

\[
((\text{input word} \mathbin{\gg} 2) \mathbin{\ll} 16)
\]

This is a 32-bit unsigned transformation. It is equivalent to taking input bits 2 through 17 and placing them in output bits 16 through 31. Input bits 0–1 and 18–31 do not contribute.

Next:

- F-000000000011 returns a word supplied to F-000000000012.
- F-000000000013 returns a word supplied to F-000000000014.
- F-000000000015, F-000000000016, and F-000000000017 are called without explicit arguments.

No additional direct memory access is shown between these calls.

### 3. Propagate the captured word to a stride-four location

F-000000000018 returns a value used to form a **second base** by multiplication by four.

The function writes the captured word at second base plus `0x14`, then calls F-000000000019 with the second base.

It next invokes F-000000000020, F-000000000021, F-000000000022, and F-000000000023 in order.

The result of F-000000000023 is shifted left by two to form a **link value**. The function writes that word to:

\[
(\text{input word} \mathbin{\&} \texttt{0xFFFFFFFC}) + \texttt{0x14}
\]

The mask discards the input word’s bottom two bits; it does not validate the resulting address. The unsigned addition is a 32-bit calculation.

F-000000000024 is then called with the link value twice, in its first two displayed argument positions.

The next sequence is:

- F-000000000025 returns a word supplied to F-000000000026.
- F-000000000027 returns a word supplied to F-000000000028.
- F-000000000029, F-000000000030, and F-000000000031 are called without explicit arguments.

### 4. Clear a word at a dependency-selected location

F-000000000032 returns a value that is multiplied by four.

The function writes a zero word at that scaled value plus `0x14`, then calls F-000000000033 with the scaled value.

It invokes F-000000000034, F-000000000035, F-000000000036, and F-000000000037 in order. The last call supplies a paired result.

Call its low 32-bit component the **pair index** and its high 32-bit component the **pair destination index**. The function shifts the pair index left by two to obtain the **shared value**.

It writes the shared value at:

\[
4 \times \text{pair destination index} + \texttt{0x14}
\]

The displayed representation treats both components as signed integers for these operations. Their permitted ranges are not established.

### 5. Snapshot a global address, then read through it after intervening calls

After the preceding store, the function reads a global word and retains it as the **saved address**.

This snapshot occurs before F-000000000038. That dependency receives four displayed arguments:

1. Shared value.
2. Pair destination index.
3. Shared value.
4. Shared value.

The function then calls F-000000000039, F-000000000040, F-000000000041, and F-000000000042 in order.

The result of F-000000000042 is multiplied by four to obtain a **third base**. The function:

1. Writes the shared value at third base plus `0x14`.
2. Reads a 32-bit word at saved address plus `0x14`.

Call the latter result the **sampled word**.

The address is snapshotted before the intervening calls, but the pointed-to word is read afterward. Consequently, dependency changes to memory at saved address plus `0x14` can affect the sampled word. A dependency changing the original global address does not, by itself, replace the saved address already captured by this body.

F-000000000043 receives the third base.

### 6. Perform the first table-directed byte store

F-000000000044, F-000000000045, F-000000000046, and F-000000000047 are called in order. The last supplies another paired result.

Call its low word the **selector word** and its high word the **companion word**. Both remain relevant after this point.

The function then:

1. Reads an unsigned byte at address `(captured word >> 23) + 1`.
2. Adds `0x11` to that byte value.
3. Writes the low byte of the sampled word to the resulting address.

The lookup address ranges from `0x1` through `0x200`. The destination address ranges from `0x11` through `0x110`.

These are direct address calculations in the supplied representation. No additional table base or object base is shown.

F-000000000048 is then called with four displayed arguments:

1. Companion word.
2. Companion word.
3. Companion word.
4. Selector word shifted left by two.

That final shift is an unsigned 32-bit operation.

### 7. Perform the second table-directed byte store

F-000000000049 returns a word supplied to F-000000000050.

The function then invokes F-000000000051, F-000000000052, F-000000000053, and F-000000000054 in order.

After those calls, it:

1. Masks the selector word with `0x3FFFFFFF`.
2. Logically shifts that result right by 21.
3. Adds one and reads an unsigned byte at the resulting address.
4. Adds `0x11` to that byte value.
5. Writes the low byte of the sampled word shifted right by 22 to the resulting address.

Thus, this store selects sampled-word bits 22 through 29. Its lookup address also ranges from `0x1` through `0x200`, and its destination ranges from `0x11` through `0x110`.

The lookup is a fresh read after intervening dependency calls. Neither table contents nor destination distinctness are guaranteed. The first byte store, or a dependency, may have changed memory used by this second lookup.

F-000000000055 receives the result of F-000000000054 shifted left by two.

### 8. Finish dependency calls and perform terminal writes

The function invokes F-000000000056, F-000000000057, F-000000000058, and F-000000000059 in order.

F-000000000060 receives the result of F-000000000059 shifted left by two, repeated in two displayed argument positions.

Next:

1. F-000000000061 is called without explicit arguments.
2. F-000000000062 returns a word.
3. That word is passed to F-000000000063.
4. F-000000000064 is called.
5. Its result is assigned to a global destination.

After that assignment, the function writes a zero byte through an implicit pointer whose origin and validity are not established.

It then writes a zero halfword at:

\[
(\text{selector word} \mathbin{\&} \texttt{0x3FFFFFFF}) \mathbin{\gg} 7
\]

This address ranges from zero through `0x7FFFFF`. No alignment adjustment or validation is shown.

Finally, the function attempts a call through address zero with three explicit zero arguments. A return from the containing function is shown only after that call.

## Side effects

### Direct memory effects

On an execution reaching every operation, the body shows:

- One initial word read and one byte store.
- Three stores of the captured word, counting the initial byte truncation separately from the two full-word stores.
- One input-addressed store of the link value.
- One explicit zero-word store.
- Two stores of the shared value.
- One global-address read and one later word read through the saved address.
- Two byte lookups, each followed by a byte store.
- One global assignment from the last dependency result.
- One zero-byte store through an implicit pointer.
- One zero-halfword store at a selector-derived address.

These describe operations, not necessarily distinct locations. Aliasing may make later writes overwrite earlier ones or alter later reads.

### Ownership and resource behavior

No ownership transfer, allocation, release, reference-count operation, or lifetime guarantee is established by the visible body.

Unknown dependencies may have such effects. Callers must not assume that passing an address-like argument borrows it, transfers it, or causes it to be released.

### Errors and partial completion

There are no visible validation branches, error checks, retries, or rollback operations.

If a dependency does not return, an address is inaccessible, an alignment requirement is violated, or the zero-address call traps, the remaining sequence is not established. Earlier effects may remain observable. No atomicity guarantee is supported.

## Dependencies

F-000000000001 through F-000000000064 are opaque, call-site-specific identifiers assigned in execution order. They identify the visible calls without asserting that all targets are distinct functions.

Their explicit arguments and consumed results are specified in Behavior. “No explicit arguments” means only that none are displayed; it does not establish a complete zero-argument ABI.

Additional distinctions:

- F-000000000001 has a target derived from incoming return-link state.
- Most other targets use fixed code-associated values with a common offset.
- F-000000000010, F-000000000012, F-000000000024, F-000000000026, F-000000000048, and F-000000000062 use differing target offsets.
- F-000000000037 and F-000000000047 provide displayed paired results.
- F-000000000064 provides the value assigned to the final global destination.
- The terminal call through address zero is a separate, explicitly null-target call.

No dependency implementation or preservation guarantee is available.

## Acceptance examples

These examples are conditional consequences of the static evidence, not executed tests. Unless otherwise stated, all preceding dependencies return, arithmetic is representable where required, and memory accesses are permitted.

1. **Initial capture and truncation.**  
   If F-000000000004 returns `0x100`, the initial read is at `0x474`. If that read yields `0x12345678`, the next direct store writes byte `0x78` at `0x401`. Later captured-word stores use `0x12345678`, even if dependencies subsequently change the original read location.

2. **Input transformation.**  
   Input word `0x00000004` produces argument `0x00010000` for F-000000000010. Input word `0x00040000` produces zero. Input word `0xFFFFFFFF` produces `0xFFFF0000`.

3. **Masked input address.**  
   Input words `0x1000`, `0x1001`, `0x1002`, and `0x1003` all select address `0x1014` for the link-value store. Those low-bit differences do not alter that store address.

4. **Delayed read through a saved address.**  
   Suppose the saved address is `0x2000`. If an intervening dependency changes the original address global to `0x3000` and writes `0xABCDEF01` at `0x2014`, the subsequent sampled-word read still targets `0x2014` and observes its contents at that later read.

5. **First byte lookup.**  
   If the captured word is `0x01800000`, the first lookup reads address `0x4`. If that byte is `0x80` and the sampled word is `0xA1B2C3D4`, the resulting store writes `0xD4` at `0x91`.

6. **Second byte extraction.**  
   If the selector word is `0x00600000`, the second lookup also selects address `0x4`. If the byte read then is `0x22` and the sampled word is `0xFFFFFFFF`, the store writes `0xFF` at `0x33`. An earlier read of address `0x4` does not fix the value of this later read.

7. **Terminal zero address.**  
   If the selector word is zero, the final halfword store targets address zero. If that access succeeds, execution still attempts the subsequent call through address zero. The visible body does not reject either operation.

8. **Interrupted execution.**  
   If F-000000000024 does not return, the link-value store has already occurred, but none of the later direct stores is guaranteed to occur.

## Uncertainties

The following remain **documented_only**:

- The intended application-level operation.
- Whether the displayed body accurately represents a conventional standalone function and its complete calling convention.
- The significance of incoming return-link state and the implicit terminal pointer.
- The semantics, memory effects, ownership behavior, and preservation properties of all opaque dependencies.
- Whether displayed paired results represent a conventional 64-bit return or another machine-level result arrangement.
- Signed-overflow behavior for several scaled dependency results and address calculations.
- Endianness, permitted unaligned accesses, address-space mapping, and memory-ordering requirements.
- Whether accesses to low addresses denote ordinary memory, mapped devices, or invalid locations in the actual environment.
- Whether the terminal zero-address call returns, traps, or has platform-specific behavior.
- The names and semantic roles of the two globals.

An inferred interpretation is that some values encode scaled references or indices. That interpretation is not required by the established behavior and must not replace the explicit calculations.

## Evidence

The supplied static representation establishes the displayed call order, argument expressions, result uses, access widths, direct memory operations, and absence of visible conditional branches.

It does not establish dependency internals, successful completion, valid memory mappings, ownership rules, or application intent. The comments and symbolic names are not treated as authoritative behavioral evidence.

No runtime tests have been performed. Acceptance examples are arithmetic and ordering consequences of the supplied static evidence, subject to the stated preconditions.
