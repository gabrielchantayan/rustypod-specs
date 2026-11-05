<!-- Trusted provenance: F-6ad14e20a3dc; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-6ad14e20a3dc

## Purpose

F-6ad14e20a3dc coordinates a conditional preparation sequence, computes a position within an object-associated region, invokes several dependencies, and returns two final 32-bit values packed into a 64-bit result.

The established behavior is the ordering of calls, address calculations, memory reads, temporary-slot updates, and result assembly. The semantic roles of the object, its indexed region, the callback-like token, and the returned pair are not established.

The function has no explicit local error return, validation, allocation, or release operation. Any such behavior inside its dependencies remains unknown.

## Interface

### Inputs

The supplied representation exposes four 32-bit arguments:

| Argument | Established use |
|---|---|
| First | No explicit use in the visible body. It is not explicitly forwarded to a dependency. |
| Second | Passed to the first invocation of F-e8bbeba61a78 when the conditional preparation path is skipped. On the taken path, replaced by a secondary register value exposed after F-8a5aadac202f returns. |
| Third | Copied into an addressable 32-bit temporary slot. That slot is exposed to dependencies and later read as an argument. |
| Fourth | Copied into a separate addressable 32-bit temporary slot. Its initial value is exposed only on the conditional path. The slot is overwritten later. |

The 32-bit widths describe the visible representation, not a proven high-level type. An argument or stored word may encode an integer, pointer, handle, or another bit pattern.

### Implicit inputs and object layout

The function obtains two values from global storage:

- An **object base**, captured near entry and retained for the initial processing sequence.
- A **control base**, used to read a 32-bit word at byte offset 4.

A zero control word skips conditional preparation. Any nonzero value enters it.

The following object-relative offsets are visible:

| Base | Byte offset | Access |
|---|---:|---|
| Captured object base | 0x54 | Read a 32-bit index-like value after conditional preparation has finished. |
| Object returned by the second F-e8bbeba61a78 invocation | 0x10 | Read a pointer-sized 32-bit word, twice at distinct points. |
| Value obtained by the first read at offset 0x10 | 8 | Read a 32-bit word. |

These offsets establish only the accessed locations. They do not establish complete object layouts, field names, allocation extents, or ownership.

### Output

The visible result is 64 bits:

- Bits 0–31 contain the final contents of the lower-result temporary slot.
- Bits 32–63 contain the final contents of the upper-result temporary slot.

Both slots are passed by address to F-8310debf75fb before the result is assembled. The returned words therefore reflect their contents **after that dependency returns**, not necessarily the values initially loaded from object memory.

This packing describes numeric bit positions. It does not establish the byte order of a subsequently stored result.

### Preconditions

For defined completion along the visible path:

- Global storage used to obtain the object and control bases must be readable.
- The control word at offset 4 must be readable.
- The captured object must support the read at offset 0x54.
- The address computed from that object and the index must be acceptable to the dependencies receiving it.
- The second F-e8bbeba61a78 invocation must produce a value suitable for the reads at offset 0x10 and the subsequent indirect read at offset 8.
- Dependencies must return normally and meet any additional requirements of their own.

No local check enforces these conditions. Alignment, address-space rules, object bounds, and dependency-specific requirements are documented_only unknowns.

## Behavior

### 1. Capture initial state

The function first captures the object base from global storage.

It copies the third and fourth arguments into two distinct, addressable 32-bit slots. These are called the **third-argument slot** and **fourth-argument slot** below.

It then reads the control word at control-base offset 4 and branches on whether that word is zero.

The captured object base is not visibly refreshed after this decision. A dependency that changes the global object-base storage does not, merely by doing so, change the already captured value used by this invocation.

### 2. Perform conditional preparation

If the control word is zero, this entire sequence is skipped. The second argument remains the value supplied by the caller.

If the control word is nonzero, the sequence is:

1. Invoke F-9b9c1c7483f4 with:
   - The address of the fourth-argument slot.
   - A fixed executable-address token.
   - The address of the third-argument slot.
2. Pass its 32-bit result, together with the captured object base, to F-17d67fcf9aa1.
3. Invoke F-8a5aadac202f with the address of the fourth-argument slot.
4. Replace the working second-argument value with the secondary register value exposed after F-8a5aadac202f returns.

The last value is not established as an ordinary declared return value. Its relationship to the caller’s second argument, the fourth-argument slot, or any resource is unknown. A contract must not silently substitute the original second argument on this path.

Both addressable slots may have been modified by dependencies before subsequent processing. In particular, the later use of the third-argument slot is not guaranteed to equal the original third argument.

The fixed executable-address token is passed onward; the visible body does not itself invoke it. Its signature and any invocation by F-9b9c1c7483f4 are unknown.

### 3. Compute an indexed address

After conditional preparation, the function reads the 32-bit word at captured-object offset 0x54.

It derives an address from:

- The captured object base.
- That word multiplied by 28 bytes.
- Addition of the resulting displacement to the captured object base.

Thus, index zero produces the object base; index one produces an address 28 bytes beyond it, provided the arithmetic is representable.

The representation treats the index and intermediate arithmetic as signed 32-bit quantities. It contains no visible range check or overflow check. The behavior of overflowing multiplication or addition is not sufficiently established to promise either mathematical unbounded arithmetic or a particular wraparound rule. Negative-index cases likewise require valid target-address semantics and valid dependency inputs; they are not locally rejected.

The index read occurs **after** all calls in the conditional sequence. Mutations to the captured object’s index field during that sequence can therefore affect the computed address.

### 4. Process the computed position

The function invokes F-e8bbeba61a78 with four explicitly represented values, in order:

1. The captured object base.
2. The working second-argument value.
3. The current contents of the third-argument slot.
4. The computed address.

It then invokes F-3a75ad37fd68 with:

1. The same computed address.
2. The 32-bit result from that F-e8bbeba61a78 invocation.

No fresh index read or address recomputation appears between these calls. If the first dependency changes the object’s index field, the second dependency still receives the address computed earlier.

The result of F-3a75ad37fd68, if any, is not explicitly consumed.

### 5. Obtain the object used for final extraction

The function invokes F-e8bbeba61a78 a second time. Only the captured object base is explicitly supplied in this call’s representation.

This invocation must not be treated as equivalent to repeating the earlier four-argument call. The true calling convention, any additional register inputs, and the reason for the differing visible arities remain unknown.

The returned 32-bit value becomes the base for final extraction. The function does not establish that it equals the originally captured object base.

### 6. Read and finalize the result pair

Using the object returned by the second invocation, the function performs these operations in order:

1. Read the 32-bit word at object offset 0x10.
2. Treat that word as an address and read a 32-bit word at offset 8 from it.
3. Store the indirectly read word in the upper-result slot.
4. Read the word at object offset 0x10 again.
5. Store this second read in the lower-result slot.
6. Copy the lower-result slot into the fourth-argument slot.
7. Invoke F-8310debf75fb with:
   - The address of the third-argument slot.
   - The returned object.
   - The address of the upper-result slot.
   - The address of the lower-result slot.
8. Assemble the 64-bit result from the final upper- and lower-result slots.

The two reads at offset 0x10 are separate observable accesses. The first determines the address for the indirect read; the second supplies the initial lower-result word. If those reads can observe different values, the pair need not describe one consistent object snapshot.

The fourth-argument slot receives a copy before F-8310debf75fb runs. It is not an alias of the lower-result slot. Subsequent changes to the lower-result slot do not automatically update that copy.

## Side effects

### Established local effects

The function writes its own temporary slots, including slots whose addresses are exposed to dependencies. It performs the global and object-relative reads described above.

The visible body contains no direct write to global storage or object fields.

### Dependency-mediated effects

The function permits dependencies to:

- Access or modify the third- and fourth-argument slots when their addresses are supplied.
- Access or modify the final result slots through the addresses supplied to F-8310debf75fb.
- Act on the captured object, returned object, and computed address.
- Potentially modify other reachable state.

The scope of these effects is unknown. Dependencies must not be assumed to preserve global state or object contents.

### Ownership and lifetime

No ownership transfer, reference-count change, allocation, destruction, or release is established by the visible body.

The call to F-8a5aadac202f follows earlier use of the fourth-argument slot, but this ordering alone does not prove cleanup semantics.

Addresses of temporary slots have invocation-local lifetime. Whether dependencies retain these addresses is unknown; safe later use is not established.

### Errors and interruption

There is no explicit local error branch, null check, retry, rollback, or status translation. If a dependency fails to return normally, subsequent calls and final result assembly are not established to occur. Invalid memory accesses are not converted into a defined error result by the visible body.

## Dependencies

| Opaque function ID | Established interaction |
|---|---|
| F-9b9c1c7483f4 | Conditional call receiving two temporary-slot addresses and a fixed executable-address token; its 32-bit result is forwarded to F-17d67fcf9aa1. |
| F-17d67fcf9aa1 | Conditional call receiving the captured object base and the preceding result. No result is explicitly consumed. |
| F-8a5aadac202f | Conditional call receiving the fourth-argument slot’s address. A secondary register value exposed afterward replaces the working second argument. |
| F-e8bbeba61a78 | Called twice: first with four explicit values, then with only the captured object base explicitly represented. The second result is used as an object address. |
| F-3a75ad37fd68 | Receives the computed address and the first F-e8bbeba61a78 result. |
| F-8310debf75fb | Receives the third-argument slot’s address, the final object, and both result-slot addresses. Final slot contents determine the return value. |

These are call-boundary observations, not semantic descriptions of the dependencies.

## Acceptance examples

These examples are conditional static consequences, not executed tests.

### Zero control word

Given readable state, a zero control word, and normally returning dependencies:

- None of the three conditional dependencies is invoked.
- The first F-e8bbeba61a78 invocation receives the caller’s second argument.
- Its third explicit value is the original third argument, absent an independently established external mutation of the temporary.
- Indexed processing and final extraction still occur.

### Nonzero control word with modified temporary

If F-9b9c1c7483f4 changes the third-argument slot to a value X, and no later action changes that slot before indexed processing, the first F-e8bbeba61a78 invocation receives X rather than the original third argument.

Its second explicit value comes from the secondary register value exposed after F-8a5aadac202f, not necessarily from the caller.

### Index changed during preparation

If the captured object’s index field changes from 2 to 5 during conditional preparation, and remains 5 until the index read, the computed displacement is 140 bytes, subject to valid address arithmetic. The earlier value 2 is not used.

### Stable extraction with unchanged result slots

Suppose the final object’s two reads at offset 0x10 both yield 0x1000, the word at address 0x1008 is 0xAABBCCDD, and F-8310debf75fb leaves both result slots unchanged. The returned value is 0xAABBCCDD00001000.

### Final dependency replaces the pair

If F-8310debf75fb leaves the upper-result slot as 0x11223344 and the lower-result slot as 0x55667788, the returned value is 0x1122334455667788, regardless of their initial extracted values.

### Unstable field reads

If the first offset-0x10 read yields address A and the second yields B, the initial upper-result word comes from A plus 8, while the initial lower-result word is B. The visible body provides no consistency check.

## Uncertainties

The following remain documented_only:

- High-level purpose, argument types, object identity, and field meanings.
- Dependency signatures and effects beyond the visible call boundaries.
- The second F-e8bbeba61a78 invocation’s full calling convention.
- Meaning and reliability of the secondary register value after F-8a5aadac202f.
- Overflow behavior, pointer validity rules, alignment requirements, and concurrency guarantees.
- Resource ownership, retention of temporary addresses, and release obligations.
- Error behavior within dependencies.
- Whether the executable-address token is invoked, and with what interface.

The 28-byte stride suggests indexed storage, and the final dependency may transform the extracted pair. These are inferred intent, not established semantic guarantees.

## Evidence

This contract is based solely on the supplied static function representation and opaque dependency mapping.

Established evidence includes the four 32-bit inputs, 64-bit result assembly, conditional call sequence, explicit argument flow, 28-byte address stride, accessed offsets, repeated field read, and address exposure of temporary slots.

No dependency implementations, runtime traces, external API contracts, or runtime tests were provided. No runtime tests have been performed.
