<!-- Trusted provenance: F-d60fc1ba809d; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-d60fc1ba809d

## Purpose

The supplied static representation describes a low-level routine that performs three coprocessor-directed operations, invokes fourteen dependency call sites in a fixed recovered order, and attempts a 16-bit memory write. The write address is derived from the upper twelve bits of a 64-bit dependency result.

No application-level purpose is established. Hardware initialization, synchronization, decoding, or device control would be inferred intent, not a supported contract.

The representation ends at an explicitly reported invalid-instruction boundary. It does not establish a normal return, a reliable complete control-flow graph, or executable behavior beyond that boundary. This contract therefore describes the recovered path conditionally: each later operation is reached only if preceding operations complete and execution follows the represented path.

## Interface

### Explicit inputs

The recovered interface has three 32-bit arguments:

| Argument | Established use |
|---|---|
| First | No explicit read appears on the recovered path. |
| Second | No explicit read appears on the recovered path. |
| Third | Supplies bit fields and shifted words to coprocessor operations and dependency calls. |

Let **P** denote the third argument, interpreted as an unsigned 32-bit word.

The absence of explicit reads of the first two arguments does not prove that their incoming register contents are unobservable to dependencies. The calling convention, dependency implementations, and completeness of the recovered argument lists are unknown.

### Additional execution inputs

The routine also consumes entry state not represented by its three arguments:

- An incoming general-purpose register value used in the first coprocessor operation.
- An incoming link-register value used to form the first dependency target.
- Two incoming coprocessor-related values used as operands in coprocessor operations.

These are real dependencies of the recovered representation. Calling the routine with only the three explicit arguments does not fully determine its behavior.

### Outputs and termination

The recovered signature has no declared return value. No ordinary return is visible.

Observable results can include:

- Coprocessor effects.
- Effects of dependency calls.
- A final attempted 16-bit memory store.
- A fault, trap, or other interruption before or at the invalid-instruction boundary.

The invalid-instruction marker is evidence of incomplete or unreliable recovery at the end of the path. It is not proof that the original executable deliberately calls an error handler.

### Arithmetic model

The description below uses 32-bit word shifts with discarded high bits, and unsigned extraction from 64-bit results. This is the useful machine-word interpretation of the recovered operations.

Some intermediate values are represented as signed integers in the supplied material. A portable language-level interpretation of their left shifts is not established. In particular, shifts that discard all significant bits must not be justified through signed-language overflow rules.

No memory ownership, allocation, or release convention is declared by this interface.

## Behavior

### 1. Initial coprocessor operations and first call

The first represented observable action is a coprocessor-directed operation with:

- Coprocessor identifier 0.
- Selector values 7 and 7.
- A data word formed by shifting the incoming hidden general-purpose register value left by 23 bits.
- The two incoming coprocessor-related values.

The meaning of those selector fields and the hardware effect are unknown.

The routine derives a word from the low two bits of P, shifted left by 16 bits. Call this word **B**. Its possible values are zero, 65,536, 131,072, and 196,608.

A second coprocessor-directed operation follows, using:

- Coprocessor identifier 0.
- Selector values 0 and 7.
- A zero data operand.
- The second incoming coprocessor-related value in both remaining represented operand positions.

The routine then calls **F-000000000001** with one explicit 32-bit argument, zero. Its target is formed from the incoming link-register value plus an offset of 3,968 bytes. Address wrapping, instruction-set state, target alignment, and target validity are not established.

Let **R** be the 32-bit result of that call.

### 2. Third coprocessor operation and first result chain

After F-000000000001 returns, the routine performs a third coprocessor-directed operation with:

- Coprocessor identifier 0.
- Selector values 2 and 7.
- P shifted left by 8 bits.
- The two represented coprocessor-related values.

The routine next calls **F-000000000002** with three 32-bit words, in order:

1. P shifted left by 8 bits.
2. P shifted left by 12 bits.
3. R shifted left by 9 bits.

No result from this call is subsequently used explicitly.

The routine calls **F-000000000003** with no explicit arguments and receives a 64-bit result. It then calls **F-000000000004** with:

1. The low 32 bits of that result.
2. The high 32 bits of that result.
3. The low 32 bits shifted left by one bit.

This division is by numeric significance, not by memory byte order. No in-memory layout for the 64-bit value is established.

### 3. Low-bit-derived transformation chain

The routine calls **F-000000000005** with B shifted left by 8 bits. Under the 32-bit word model, this argument equals the low two bits of P shifted left by 24 bits. Only bits 24 and 25 can be set.

Its 32-bit result is passed to **F-000000000006**. That dependency’s 32-bit result is then passed to **F-000000000007**.

The routine next calls **F-000000000008** with no explicit arguments. No result from this call is used explicitly.

This sequence establishes data flow and ordering only. It does not establish whether these dependencies transform values, perform I/O, mutate shared state, or retain references.

### 4. Address-producing result chain

The routine calls **F-000000000009** with no explicit arguments and receives a 64-bit result.

It calls **F-00000000000a** with three 32-bit words:

1. The low 32 bits of the preceding result.
2. Its high 32 bits.
3. Its high 32 bits shifted left by 11 bits.

Let **V** be the 64-bit result returned by F-00000000000a.

Two portions of V subsequently matter:

- Its low 32 bits are passed to **F-00000000000b**.
- Its upper bits determine the final store address.

The recovered arithmetic first shifts V right by 41 bits and retains the result as a 32-bit unsigned word. This intermediate is at most 23 bits wide. Consequently, converting it to a signed 32-bit value does not make it negative, and the later signed right shift by 11 bits has no sign-extension ambiguity.

The resulting value, **Q**, is therefore V shifted right by 52 bits. Q ranges from 0 through 4,095.

The eventual store address is:

**A = Q + floor(Q / 4)**

A ranges from 0 through 5,118 inclusive. There is no represented base address, bounds check, null check, alignment adjustment, or permission check.

### 5. Final dependency calls

Before the final store, the routine shifts B left by 25 bits.

Under the 32-bit word model, this produces zero for every possible P: B contains significant bits only at positions 16 and 17, and shifting those positions left by 25 discards them both. This zero result is separate from the earlier nonzero-capable argument to F-000000000005.

Following F-00000000000b, the routine:

1. Calls **F-00000000000c** with no explicit arguments.
2. Calls **F-00000000000d** with the 32-bit result of F-00000000000c and the shifted B word.
3. Calls **F-00000000000e** with no explicit arguments.

Thus, under the stated word model, the second explicit argument to F-00000000000d is always zero.

These calls occur after V has been obtained and before the memory write. They may affect memory mappings, permissions, hardware state, or the destination contents. Their implementations are unavailable, so none of those states may be assumed unchanged.

### 6. Store and terminal boundary

If execution reaches the final store, the routine attempts to write the 16-bit value **0x00CD** at address A.

The representation specifies a halfword-sized store, not a pair of independently ordered byte stores. Byte order, bus transactions, and atomicity are unknown.

After that store, the representation terminates at an invalid-instruction marker. No further source-visible operation or ordinary return is established.

## Side effects

### Ordering

On an uninterrupted traversal of the recovered path:

- The first two coprocessor operations precede F-000000000001.
- The third coprocessor operation follows that call and precedes F-000000000002.
- The fourteen dependency calls occur in the order listed above.
- F-00000000000e precedes the final store.

This is recovered operation order. It does not establish hardware memory barriers, completion of posted device writes, cache coherence, or ordering of effects internal to dependencies.

### Memory access

The only explicit ordinary-memory store is the final 16-bit write.

Its start address can be zero, odd, or even. For example, Q equal to 1 produces address 1. The represented path does not reject or repair an unaligned destination.

For ordinary byte-addressed memory, a two-byte store starting at the maximum address would cover addresses 5,118 and 5,119. Whether that access is permitted or has ordinary RAM semantics is unknown.

No explicit ordinary-memory load is shown outside dependency invocation and target acquisition.

### Ownership and shared state

No explicit allocation, release, ownership transfer, or pointer lifetime management is established.

Do not assume dependencies preserve globals, registers outside the applicable calling convention, device state, or memory mappings. Likewise, do not assume they allocate or release memory merely because their implementations are unavailable.

### Errors

There is no represented status check, retry, rollback, or recovery branch.

If a dependency fails to return, the remaining path is not reached. If a coprocessor operation, indirect call, or memory store faults, no local recovery behavior is established.

Successful completion of earlier operations does not guarantee that the final store occurs.

## Dependencies

The following opaque IDs are document-local labels for distinct recovered call sites. They do not assert globally verified dependency identities or prove that different call sites reach different implementations.

| ID | Explicit arguments | Result used |
|---|---|---|
| F-000000000001 | One zero word | 32-bit word |
| F-000000000002 | Three shifted words | None |
| F-000000000003 | None | 64-bit value |
| F-000000000004 | Low word, high word, shifted low word | None |
| F-000000000005 | Low-two-bit-derived word | 32-bit word |
| F-000000000006 | Previous result | 32-bit word |
| F-000000000007 | Previous result | None |
| F-000000000008 | None | None |
| F-000000000009 | None | 64-bit value |
| F-00000000000a | Low word, high word, shifted high word | 64-bit value |
| F-00000000000b | Low word of V | None |
| F-00000000000c | None | 32-bit word |
| F-00000000000d | Previous result and shifted B | None |
| F-00000000000e | None | None |

“None” under arguments means no explicit arguments are represented, not absence of hidden state. “None” under result means no returned value is explicitly consumed.

## Acceptance examples

These are conditional static acceptance examples, not executed tests. They assume the recovered path is executable, preceding calls return normally, and shifts use the stated word model.

1. **Low-bit-derived arguments.**  
   For P equal to 3, F-000000000005 receives **0x03000000**. F-00000000000d nevertheless receives zero as its second argument. Treating those two arguments as the same value would contradict the recovered arithmetic.

2. **Truncation of shifted P.**  
   For P equal to **0xFFFFFFFF**, the first two arguments to F-000000000002 are **0xFFFFFF00** and **0xFFFFF000**, respectively. High bits shifted out of a 32-bit word are not retained.

3. **Zero destination.**  
   If the upper twelve bits of V are zero, A is zero regardless of V’s lower 52 bits. The routine attempts the halfword write at address zero; the representation contains no null-address guard.

4. **Unaligned destination.**  
   If V has Q equal to 1, A is 1. An alignment fault is a possible platform outcome, not a represented validation branch.

5. **Quarter-step address adjustment.**  
   Q equal to 3 produces A equal to 3. Q equal to 4 produces A equal to 5. The address rule is not a constant stride applied directly to Q.

6. **Maximum destination.**  
   If V’s upper twelve bits are all set, Q is 4,095 and A is 5,118. Lower bits of V do not alter that address.

7. **Lower-word consumption remains distinct.**  
   Two V values with identical upper twelve bits but different low words produce the same A, while F-00000000000b receives different arguments. Equal store addresses do not imply otherwise identical behavior.

8. **Interrupted execution.**  
   If F-00000000000e does not return, the final store is not established as occurring. If it returns after changing destination permissions, the store may fault.

## Uncertainties

The following remain **documented_only**:

- Whether the recovered path accurately represents executable instructions throughout.
- The actual calling convention and completeness of argument recovery.
- Coprocessor semantics, privilege requirements, and ordering guarantees.
- The hidden entry-register values and validity of the first computed target.
- Dependency implementations, side effects, aliasing, and termination behavior.
- Exact executable semantics of the signed-looking shift expressions.
- Address-space interpretation, byte order, alignment rules, and store atomicity.
- Whether the final address denotes RAM, device memory, unmapped space, or another platform-specific region.
- Actual execution behavior at and beyond the invalid-instruction boundary.

No inferred application intent is required to explain the established data flow.

## Evidence

The evidence is limited to the supplied static representation: a three-argument interface, three coprocessor-directed operations, fourteen dependency call sites, explicit word and double-word data flow, one 16-bit store, and an invalid-instruction termination warning.

The zero-valued final shifted argument and the address range are arithmetic deductions under the stated 32-bit word model. They are not runtime observations.

No implementation inspection beyond the supplied material, dependency inspection, runtime execution, or tests have been performed.
