<!-- Trusted provenance: F-b5a0a3ae8c31; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-b5a0a3ae8c31

## Purpose

The available static evidence describes an ordered sequence of indirect dependency calls, one 32-bit memory write, and four byte writes. The sequence uses three explicit inputs, two additional entry-state values, and several dependency results to form arguments and memory addresses.

No application-level purpose is established. In particular, the evidence does not justify describing this function as an allocator, serializer, device initializer, memory copier, or resource-release operation. Repeated argument values, shifts, and small destination addresses are insufficient to establish any of those interpretations.

This contract describes the visible straight-line prefix. The supplied representation reports invalid instruction data and ends at an invalid-instruction boundary rather than a normal return. Execution of the complete prefix is therefore conditional on the representation being accurate, every preceding dependency returning, and every preceding memory access completing.

## Interface

### Explicit inputs

The visible interface has three explicit inputs:

| Input | Visible width | Established use |
|---|---:|---|
| First input | 32 bits | Passed twice to the first dependency and later stored as a 32-bit memory value. |
| Second input | Unsigned 32 bits | Logically shifted right by three bits and passed to the first dependency. |
| Third input | 32 bits | Passed unchanged to the first dependency. |

The first and third inputs have no established semantic type. They may contain integers, addresses, handles, or other bit patterns; the evidence does not distinguish these possibilities.

The second input’s right shift discards its three least significant bits. There is no visible rounding upward, remainder handling, range validation, or rejection of values not divisible by eight.

### Additional entry state

Two values are consumed without being established by the explicit interface:

- An **entry control-address value**, used to derive the first indirect call target.
- An **entry memory-base value**, used twice to locate a selector byte at offset one.

These are dependencies on incoming machine state, not demonstrated additional formal parameters. Their initialization, validity, and relationship to the explicit inputs are unknown.

### Output and completion

No normal return value is established. Although the visible interface is represented without a return value, the recovered prefix does not reach an ordinary return.

Observable results are instead:

- Dependency invocations and their arguments.
- A 32-bit memory write.
- Four byte writes, including two writes of zero.
- Any effects produced by the dependencies.
- Arrival at the invalid-instruction boundary if the entire visible prefix completes.

The boundary is not evidence of a deliberate, recoverable error return or a named exception API.

### Width and address conventions

The visible values used for word arithmetic and addresses are 32 bits. Selector reads and selector-directed writes are eight bits. One dependency result is represented as 64 bits and split into lower and upper 32-bit halves.

Where the behavior below describes wraparound, it uses a **32-bit machine-word interpretation**: multiplication and left shifts retain the low 32 bits. This interpretation makes the address and argument relationships explicit, but the supplied high-level representation alone does not establish every instruction’s treatment of signed overflow or invalid signed shifts. Those limitations are documented under **Uncertainties**.

No byte order, alignment policy, memory permissions, or calling convention is established.

## Behavior

### 1. Initial dependency calls

1. Invoke **F-000000000001** with four visible arguments, in this order:
   - First input.
   - First input again.
   - Third input.
   - Second input logically shifted right by three bits.

   The target depends on the entry control-address value. No result from this call is visibly consumed.

2. Invoke **F-000000000002** with no explicit arguments shown. Treat its visible result as a 64-bit quantity.

3. Invoke **F-000000000003** with:
   - The lower 32 bits of that result.
   - The upper 32 bits.
   - The lower 32 bits again.
   - The upper 32 bits logically shifted right by three bits.

The lower half is forwarded as a 32-bit bit pattern. A signed-looking conversion in the supplied representation does not establish a signed application-level meaning.

### 2. First word write

4. Invoke **F-000000000004**, **F-000000000005**, and **F-000000000006**, in that order. No explicit arguments or consumed results are shown.

5. Invoke **F-000000000007** and consume its 32-bit result as an address-scaling value.

6. Write the complete first input as a 32-bit value at the address obtained by multiplying that result by four and adding twenty.

   Under the 32-bit machine-word interpretation, both operations wrap at 32 bits. There is no visible bounds check, null check, alignment check, or validation of the returned value before the write.

7. Invoke **F-000000000008** with the result from step 5 multiplied by four. This argument excludes the additional twenty used for the write address.

The word write occurs before this dependency invocation. The evidence does not establish that the dependency receives an allocation base, owns the stored value, or can access the destination safely.

### 3. Capture a word and perform the first selector-directed write

8. Invoke **F-000000000009**, **F-00000000000a**, and **F-00000000000b**, in order, without explicit arguments shown.

9. Invoke **F-00000000000c** and retain its unsigned 32-bit result as the **captured word**.

10. Form the **scaled word** by shifting the captured word left by two bits, retaining its low 32 bits under the machine-word interpretation.

11. Read one unsigned byte from the entry memory-base value plus one.

12. Add seventeen to that byte’s numeric value and write one byte at the resulting address. The written byte is the low eight bits of the scaled word.

The selector is the byte value itself, not a pointer-sized value read from memory. Its destination is therefore numerically between seventeen and 272 inclusive. No additional pointer dereference is shown between adding seventeen and performing the write.

The stored value depends only on captured-word bits zero through five. Its low two bits are always zero.

### 4. Perform the second selector-directed write

13. Invoke **F-00000000000d**, **F-00000000000e**, and **F-00000000000f**, in order. Retain the 32-bit result of the last call.

14. Read the byte at the entry memory-base value plus one again. Do not substitute the earlier selector value.

15. Add seventeen to this newly read byte and write one byte there. Compute the value by:
   - Retaining the low thirty bits of the captured word.
   - Logically shifting them right by twenty bits.
   - Retaining the low eight bits for the store.

The mask and shift produce at most ten bits before the byte truncation. Consequently, the stored byte contains captured-word bits twenty through twenty-seven; bits twenty-eight and twenty-nine do not survive the store.

16. Invoke **F-000000000010** with the result retained in step 13 shifted left by two bits.

The intervening dependencies may change the selector memory. The second write is therefore not established to target the same address as the first write.

### 5. First derived-selector clear and four-argument call

17. Invoke **F-000000000011**, **F-000000000012**, and **F-000000000013**, in order.

18. Invoke **F-000000000014** and retain its 32-bit result.

19. Read one unsigned byte at the address formed by multiplying the scaled word by 1,024 and adding one.

20. Add seventeen to the byte just read and write a zero byte at that destination.

Under 32-bit wraparound, the selector-read address is equivalently the captured word shifted left by twelve bits, plus one. This equivalence does not change the access width: exactly one byte is visibly read.

21. Invoke **F-000000000015** with four visible arguments:
   - The scaled word shifted left by three bits.
   - The same shifted value again.
   - The scaled word.
   - The result retained in step 18 shifted left by two bits.

Under the machine-word interpretation, each of the first two arguments is also the captured word shifted left by five bits, retaining the low 32 bits.

### 6. Second derived-selector clear and final calls

22. Invoke **F-000000000016**, **F-000000000017**, and **F-000000000018**, in order.

23. Invoke **F-000000000019** and retain its 32-bit result.

24. Read the selector byte again from the address formed by multiplying the scaled word by 1,024 and adding one.

25. Add seventeen to this newly read byte and write a zero byte at the resulting destination.

The selector-read address is derived from the same retained scaled word as before. The selector value need not be unchanged: dependencies or overlapping writes may have modified the byte at that address.

26. Invoke **F-00000000001a** with the result retained in step 23 shifted left by two bits.

27. Invoke **F-00000000001b**, **F-00000000001c**, **F-00000000001d**, and **F-00000000001e**, in that order.

28. The visible representation then reaches an invalid-instruction boundary. No subsequent behavior or normal return is established.

## Side effects

### Direct memory effects

If all preceding operations complete, the visible prefix performs these direct writes in order:

| Write | Width | Destination | Value |
|---|---:|---|---|
| First | 32 bits | Four times the result of F-000000000007, plus twenty | First input |
| Second | 8 bits | First entry-base selector byte, plus seventeen | Low byte of the scaled word |
| Third | 8 bits | Re-read entry-base selector byte, plus seventeen | Captured-word bits twenty through twenty-seven |
| Fourth | 8 bits | First derived selector byte, plus seventeen | Zero |
| Fifth | 8 bits | Re-read derived selector byte, plus seventeen | Zero |

There are four explicit selector-byte reads: two using the entry memory-base value and two using the scaled-word-derived address.

These descriptions do not assert that the five destinations are distinct. A word write can overlap a later selector read or byte destination. Byte writes can overlap each other or their own selector storage. Intervening dependencies can also modify any accessible memory. Consequently, the visible writes alone do not determine final memory contents after the entire prefix.

### Ownership and lifetime

No allocation, transfer of ownership, reference counting, or release is established.

Passing the same value twice does not prove shared ownership. Passing a scaled address-like value does not prove that a dependency owns, frees, or preserves the associated storage.

The function directly attempts writes through computed addresses. A caller or execution environment must supply accessible state if those accesses are expected to complete. No visible mechanism establishes that accessibility.

### Errors and partial execution

No visible branch handles dependency failure, invalid addresses, misalignment, arithmetic overflow, or invalid entry state.

A dependency may fail to return, fault, or alter process state. A memory access may fail according to the target environment. In such cases, only the preceding completed effects are supported; there is no visible rollback.

The invalid-instruction boundary is separate from these earlier possible failures. Its actual runtime handling is unknown.

## Dependencies

The following opaque IDs distinguish the thirty visible call sites. Only the primary function has a supplied identity mapping. The dependency IDs below are contract-local labels for unresolved targets, not claims that external function identities have been recovered. Different labels do not prove different underlying callees.

| Opaque dependency ID | Visible arguments | Consumed result |
|---|---|---|
| F-000000000001 | Four input-derived words | None shown |
| F-000000000002 | None shown | 64-bit result |
| F-000000000003 | Four words derived from that result | None shown |
| F-000000000004–F-000000000006 | None shown | None shown |
| F-000000000007 | None shown | 32-bit address-scaling value |
| F-000000000008 | Address-scaling value multiplied by four | None shown |
| F-000000000009–F-00000000000b | None shown | None shown |
| F-00000000000c | None shown | Captured unsigned word |
| F-00000000000d–F-00000000000e | None shown | None shown |
| F-00000000000f | None shown | 32-bit value |
| F-000000000010 | Previous consumed result shifted left by two | None shown |
| F-000000000011–F-000000000013 | None shown | None shown |
| F-000000000014 | None shown | 32-bit value |
| F-000000000015 | Four scaled words | None shown |
| F-000000000016–F-000000000018 | None shown | None shown |
| F-000000000019 | None shown | 32-bit value |
| F-00000000001a | Previous consumed result shifted left by two | None shown |
| F-00000000001b–F-00000000001e | None shown | None shown |

“None shown” concerns explicit arguments or consumed results in the supplied representation. It is not evidence that a dependency has no implicit inputs, cannot return information through memory, or preserves global state.

## Acceptance examples

These are conditional behavioral criteria derived from static evidence, not executed tests. Each assumes that all earlier dependencies return and all required accesses succeed.

1. **Discarding low input bits.** If the second input is fifteen, the fourth visible argument to F-000000000001 is one. If it is sixteen, that argument is two. Neither case establishes a byte count or other unit.

2. **Splitting the 64-bit result.** If F-000000000002 returns the bit pattern `0x1234567889abcdef`, F-000000000003 receives `0x89abcdef`, `0x12345678`, `0x89abcdef`, and `0x02468acf`, in that order.

3. **Word destination and subsequent argument differ.** If F-000000000007 returns three, the direct 32-bit write targets address 32. F-000000000008 subsequently receives twelve, not 32.

4. **Distinct byte slices.** If the captured word is `0x12345678`, the scaled word is `0x48d159e0` under 32-bit wrapping. The first selector-directed byte write stores `0xe0`; the second stores `0x23`. The second is not a full ten-bit field write.

5. **Selector re-reading matters.** If the first entry-base selector read produces five, the first byte destination is 22. If an intervening dependency changes the selector to nine, the second byte destination is 26. Reusing destination 22 would not match the visible read order.

6. **Derived address truncation.** With the same captured word `0x12345678`, the derived selector-read address is `0x45678001` under the 32-bit machine-word interpretation. A wider untruncated product would identify different memory and is not interchangeable.

7. **Repeated clearing can target different bytes.** If the first derived selector read produces 255, the first zero write targets 272. If the later read produces zero, the second zero write targets seventeen.

8. **Failure leaves a prefix.** If the first direct word write faults, no later dependency invocation is established. If every visible operation completes, acceptance does not require a normal return; the available representation instead ends at the invalid-instruction boundary.

## Uncertainties

### Established static evidence

The supplied representation establishes the visible call order, explicit argument relationships, result consumption, access widths, selector re-reads, and terminal control-flow limitation, subject to the representation’s accuracy.

### Inferred intent

Multiplication by four and repeated shifts could reflect scaled indices or packed fields. The selector-directed writes could interact with a special memory region. These are possible interpretations only and are not contractual semantics.

### Documented-only unknowns

- Actual identities, signatures, side effects, and return guarantees of every dependency.
- Calling convention and the meaning of entry-state values.
- Whether the recovered prefix accurately represents executable instructions throughout.
- Exact machine behavior for signed-looking multiplication and left shifts in the representation.
- Pointer width beyond the visible 32-bit address calculations, address mapping, alignment requirements, and endianness.
- Whether small numeric destinations are ordinary memory, mapped device state, invalid addresses, or evidence of imperfect recovery.
- Asynchronous mutation and concurrency guarantees.
- Runtime handling of the invalid-instruction boundary.
- Any behavior beyond the visible prefix.

These unknowns must not be replaced with assumptions that dependencies preserve globals, leave selectors unchanged, or acquire or release memory ownership.

## Evidence

This contract uses only the supplied static representation of F-b5a0a3ae8c31.

The supporting observations are thirty visible indirect calls; explicit 32-bit input and intermediate values; one consumed 64-bit result; one direct 32-bit store; four selector-byte reads; four direct byte stores; and termination of the available representation at invalid instruction data.

No implementation inspection beyond that supplied material, runtime execution, memory trace, or test has been performed. The acceptance examples are calculated conditions for later verification, not reports of observed behavior.
