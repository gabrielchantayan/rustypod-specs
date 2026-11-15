<!-- Trusted provenance: F-da7994ffdb4c; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-da7994ffdb4c

## Purpose

F-da7994ffdb4c is a state-mutating routine that interleaves numerous indirect calls with byte, halfword, and word memory accesses. It propagates dependency-produced values into computed storage locations, extracts upper bits from words, updates byte-addressed state, and finally attempts a call to address zero.

The supplied static representation does not establish an application-level purpose. In particular, it does not establish that this is initialization, serialization, resource management, or a valid independently callable routine. Its unusual entry-state dependencies, low-address accesses, and indirect control transfers prevent a reliable higher-level interpretation.

This contract distinguishes:

- **Established static evidence:** operations and ordering explicitly represented in the supplied material.
- **Conditional interpretation:** consequences under a conventional 32-bit machine-word interpretation.
- **documented_only unknowns:** behavior requiring dependency bodies, calling-convention information, or executable evidence.

No runtime tests have been performed.

## Interface

### Explicit input and output

The declared interface has one 32-bit input and no declared return value.

The explicit input is passed to the first indirect dependency. The visible body does not subsequently use that input directly. This does not establish that its effects are confined to that call: the dependency may alter memory, registers, or control flow.

There is no explicit success result, error result, or returned ownership token.

### Hidden entry and call-result inputs

The routine additionally depends on state not represented as ordinary parameters:

- An incoming control-link value used to determine the first call target.
- An incoming register-sized base value used for a byte store and a word read.
- An incoming 32-bit value written to two computed word locations.
- Several secondary-register outputs from indirect calls.
- Primary results represented as either 32-bit words or 64-bit pairs.
- Mutable global storage and memory at very small numerical addresses.

These hidden inputs are not optional under the supplied representation. A conventional caller supplying only the explicit argument is not enough to establish safe execution.

### Width and address conventions

The visible memory widths are:

| Operation | Width |
|---|---:|
| Byte read or write | 8 bits |
| Final clearing write | 16 bits |
| Word read or write | 32 bits |
| Selected dependency result | 64 bits, subsequently split into two 32-bit halves |

For clarity, this contract uses these mathematical descriptions:

- **Word slot of an index:** four times the index, plus 20.
- **Sixteen-stride slot of an index:** sixteen times the index, plus 20.
- **Byte-selected destination:** read an unsigned byte at the specified selector address, add 17 to that byte’s numeric value, and use the result as the destination address.
- **Low byte:** the least-significant eight bits of a value.

These are descriptions of actual address calculations, not evidence of arrays, object headers, or allocated records.

Unsigned right shifts have zero-fill semantics in the supplied representation. Left shifts and multiplications involving signed-looking values require qualification: their intended machine behavior is not established solely by the recovered type annotations. Where examples use wrapping arithmetic, they explicitly assume 32-bit machine-word behavior.

Endianness is not established. A word read followed by storing its low byte is a numeric truncation, not necessarily a copy of the byte at the word’s lowest address.

## Behavior

### Execution model

The visible body is straight-line: there are no explicit loops, conditional branches, bounds checks, null checks, or error branches.

That does **not** guarantee completion. Every indirect call may fail, transfer control elsewhere, or not return. Every memory access requires the addressed location to support the specified access. The final zero-address call is especially significant.

The stages below preserve the order of visible data effects. Calls between those effects remain observable and must not be removed merely because their returned values appear unused. Dependencies may change memory or expose side effects.

### 1. Entry dispatch and initial stores

The routine first calls a target derived from the incoming control-link value, passing the explicit 32-bit input. Four further indirect calls follow; the last supplies a 64-bit result.

It then:

1. Stores the low byte of the incoming base value at that base plus 9.
2. Reads a word at the same base plus 52.
3. Calls a dependency with:
   - The low 32-bit result half shifted left by two.
   - The high 32-bit result half.
   - The word just read.

Four further indirect calls follow.

A secondary-register result then supplies an index-like value. The routine writes the incoming 32-bit hidden value to that value’s word slot. It next reads a word at the **unscaled** secondary value plus 20.

The distinction matters: the write uses four-times scaling; the read does not.

The secondary value is passed to a dependency. Subsequent calls include forwarding one 32-bit result into another call.

### 2. First chain of word propagation

After another call sequence, a primary 32-bit result selects a word slot. The incoming hidden 32-bit value is written there.

The word captured in the preceding stage is shifted right by 22; a word is read at that shifted value plus 20. This address is not multiplied by four.

A dependency receives four times the primary result that selected the preceding write.

After further calls, another primary result is obtained. The most recently loaded word determines the next two accesses:

1. Shift that loaded word right by one.
2. Write the new primary result shifted left by two to the resulting word slot.
3. Read another word at the original loaded word plus 20.
4. Pass the right-shifted index to a dependency.

The newly read word is retained across subsequent calls.

A later primary result selects another word slot, receiving the same shifted primary value used in the preceding write. The routine then snapshots a 32-bit global, designated **global word A**, before making more calls.

### 3. Upper-bit extraction and two zero-valued word stores

After the next call sequence:

1. The retained word from stage 2 is shifted right by 22.
2. The snapshot of global word A is shifted right by 23.
3. A secondary-register result selects a word slot, which receives the first extracted value.
4. A dependency receives the secondary result twice, followed by the two extracted values in the order: global-derived value, retained-word-derived value.

A subsequent dependency produces a 32-bit result that is forwarded to another call.

The global-derived value is shifted right by 23 again. Because the original global read is 32 bits, the first shift leaves at most nine significant bits. The second shift therefore produces zero. This conclusion does not require assuming that dependencies preserve the global: the calculation uses the already captured value.

Later:

- A primary result selects another word slot receiving the retained-word-derived value.
- A secondary-register result selects a word slot receiving zero.
- After more calls, a primary result selects a word slot receiving the twice-shifted global-derived value, also zero.

Before proceeding, the routine reads a word at:

- The secondary result masked to its low 29 bits,
- Then shifted right by 19,
- Then increased by 20.

The masked-and-shifted address component is between 0 and 1023.

### 4. Low-address byte selection and clearing

The word just read is shifted right by 22, yielding a value between 0 and 1023.

The routine reads a byte at the twice-shifted global-derived value plus 1. Since that value is zero, this is a byte read at numerical address 1. The byte read selects a destination between addresses 17 and 272. The low byte of the shifted word is stored there.

A dependency is called with the same secondary-register value in its first three visible argument positions and zero in its fourth. After more calls, another dependency receives zero.

Later, the routine:

1. Reads a byte at four times the extracted 10-bit value, plus 1.
2. Writes zero to the byte-selected destination.
3. Passes a newly returned primary value shifted left by two to a dependency.
4. Obtains another primary value and passes its left-shifted form twice to a dependency.
5. Obtains a 32-bit result and forwards it to another dependency.
6. Makes another call.
7. Reads the byte at address 1 again and clears the selected destination.

The second read at address 1 is a fresh read. It must not be replaced with the earlier byte, because intervening dependencies and writes may have changed memory.

### 5. Repeated low-address clearing and global publication

A secondary-register result is captured and retained for later shifts. A dependency receives that captured value in three visible argument positions.

After another result-forwarding sequence, a dependency receives both a new 32-bit result and the prior stage’s 32-bit result shifted right by 23.

Following more calls, the routine:

1. Reads the byte at address 1 again.
2. Saves its zero-extended numeric value.
3. Clears the byte at that saved value plus 17.
4. Calls a dependency with a newly returned primary value shifted left by two.
5. Obtains another primary value and calls a dependency with its shifted value twice.
6. Obtains a 32-bit result.
7. Stores that result into **global word B**.
8. Calls a dependency with that result.
9. Makes another call.
10. Reads a word at the saved byte value plus 116.
11. Stores that word’s low byte at the saved byte value plus 1.

The saved byte value, not a new read of address 1, determines the final two addresses in this stage.

### 6. Word-to-byte copying at scaled bases

A dependency receives the retained secondary value shifted left by 11. Another supplies a 64-bit result. The next visible call receives its low half, high half, and the stage-5 published result shifted right by 23.

After further calls, a primary result multiplied by four becomes a byte-copy base:

- Read a word at that base plus 116.
- Store its low byte at that base plus 1.
- Pass the base to a dependency.

More calls produce a second primary result. Its left-shifted-by-two form is retained separately.

The routine then repeats the word-read/byte-write operation at the **previous** byte-copy base. This is a fresh read after intervening calls; it is not reuse of the earlier loaded word.

A dependency next receives:

- The retained secondary value shifted left by 12.
- The new shifted primary value twice.
- The same new primary value shifted left by 18.

Two further result-forwarding transitions follow.

### 7. Word replication and linked writes

A later primary result multiplied by four selects another base. The routine reads a word at that base plus 116, saves the full word, and stores its low byte at the base plus 1.

After a call sequence:

1. A new primary result selects a sixteen-stride slot.
2. The saved full word is written there.
3. A dependency receives the stage-6 shifted primary value, logically shifted right by two and then shifted left by 16.
4. Result-forwarding calls follow.
5. Another primary result selects a word slot receiving the saved full word.

More calls produce a primary result whose left-shifted-by-two form is stored at the stage-6 retained shifted value, with its low two bits masked off, plus 20. The retained shifted value already has two low zero bits under ordinary 32-bit shift interpretation; the mask does not make the address validated.

A dependency receives the newly shifted value twice. Further result-forwarding calls lead to:

- A zero write to a new primary-result-selected word slot.
- A dependency call with four times that primary result.
- Another call sequence yielding a 64-bit result.

The low half of that 64-bit result, shifted left by two, is written to the word slot selected by its high half.

The routine then reads global word A again. This is a new snapshot, independent of the earlier snapshot.

A dependency receives the shifted low half, the high half, and the shifted low half twice more. Later calls produce another primary result selecting a word slot that receives the same shifted low half.

Finally, a word is read at the newly captured global word A value plus 20.

### 8. Final byte updates and termination path

After further calls, another 64-bit result is split into low and high halves.

The routine performs these byte updates in order:

1. Read a selector byte at the stage-7 saved full word shifted right by 23, plus 1.
2. Store the low byte of the final word read in stage 7 to that selector’s destination.
3. Call a dependency with the latest high half three times and the latest low half shifted left by two.
4. Perform result-forwarding and additional calls.
5. Read another selector byte at the latest low half masked to 30 bits, shifted right by 21, plus 1.
6. Store the low byte of the stage-7 final word shifted right by 22 to that selector’s destination.

Subsequent dependencies receive newly produced values shifted left by two, first once and then twice. Another dependency supplies a 32-bit result for forwarding. The final call in this sequence supplies an 8-bit result and a secondary-register pointer.

The terminal effects are:

1. Write zero through that secondary-register pointer.
2. Replace global word B with a 32-bit concatenation whose low byte is the final 8-bit result and whose upper 24 bits come from a separate represented global fragment.
3. Write a zero halfword at the latest low 32-bit result, masked to 30 bits and shifted right by seven.
4. Attempt an indirect call to address zero with three visible zero arguments.
5. Reach the declared return only if that call returns.

The halfword destination is not explicitly aligned. Its calculation can produce an odd address.

## Side effects

### Memory and aliasing

All listed stores are externally observable if execution reaches them. No rollback is visible.

Computed destinations may alias:

- Earlier destinations.
- Selector bytes read later.
- Global storage.
- Dependency-accessed state.
- Low-address memory.

Consequently, writes and fresh reads must retain their visible order. A value saved before a call is distinct from a later reread of the same global or memory address.

The static material warns of overlapping global representations. Global word B and the upper-24-bit fragment must therefore not be assumed disjoint. The final concatenation is not proven to preserve the upper bits of the earlier value stored to global word B.

### Ownership and lifetime

No allocation, release, ownership transfer, or resource lifetime protocol is established.

Unknown dependencies may allocate, release, retain, or invalidate memory. The routine’s reuse of values across calls does not prove that those values remain valid pointers.

### Errors and partial execution

There is no visible error normalization or recovery. Potential failure points include invalid targets, unmapped memory, unsupported low-address access, alignment faults, invalid hidden register state, and nonreturning dependencies.

Any preceding stores may remain visible when a later operation fails. Atomicity, synchronization, and thread safety are not established.

## Dependencies

The supplied dependency map identifies only F-da7994ffdb4c itself. It does not identify the callees. The following opaque IDs classify visible call-target constructions; they do not assert that all sites in one class call the same function.

| Opaque ID | Established dependency form |
|---|---|
| F-00000001 | Initial target derived from incoming control-link state; receives the explicit argument. |
| F-00000002 | Dominant site-relative indirect-call family; results include words, wide pairs, and secondary-register outputs. |
| F-00000003 | Alternate site-relative family used after selected word writes. |
| F-00000004 | Alternate family with repeated visible arguments around byte updates. |
| F-00000005 | Site receiving the retained value shifted left by 11. |
| F-00000006 | Site receiving the four shift-derived arguments in stage 6. |
| F-00000007 | Alternate result-forwarding site following that call. |
| F-00000008 | Alternate family used in stage 7, including a repeated shifted-value call. |
| F-00000009 | Alternate result-forwarding family following those calls. |
| F-0000000a | Alternate site receiving the right-shifted-then-left-shifted stage-6 value. |
| F-0000000b | Late alternate site producing a forwarded 32-bit result. |
| F-0000000c | Final target at address zero; receives three visible zeros. |

“No explicit arguments” at a call site does not prove that no register inputs are consumed. The argument lists described above likewise establish only the operands visible in the supplied representation.

No dependency is assumed to preserve globals, memory, or ownership.

## Acceptance examples

These examples are conditional static expectations, not executed tests.

1. **Initial truncation:** If the hidden base is 4096 and the destination is writable, the initial byte store writes zero at address 4105. It does not write the explicit input there.

2. **Scaled write versus unscaled read:** If the first secondary result is 12, the hidden incoming word is written at address 68. The following word read is at address 32.

3. **Double extraction:** For any 32-bit snapshot of global word A, shifting right by 23 and then right by 23 yields zero. The later store using that value writes zero, provided execution reaches it.

4. **Ten-bit extraction and truncation:** If the stage-3 masked-address read yields all one bits, its right shift by 22 yields 1023. The subsequent byte store writes 255, not 1023.

5. **Selector indirection:** If a selector-byte read yields 40, the destination is address 57. It is not the selector’s own address plus 17.

6. **Fresh selector reads:** If a dependency changes the byte at address 1 between two represented reads, the later read observes the changed value under ordinary sequential-memory assumptions. Reusing the first selector would violate the contract.

7. **Fresh word-to-byte copy:** If an intervening dependency changes the word at a retained base plus 116, the repeated copy uses the changed word’s low byte.

8. **Global publication order:** The stage-5 word result is stored to global word B before the subsequent dependency receives it. A later final concatenation replaces that global again.

9. **Unaligned terminal destination:** If the latest wide-result low half is 128, the final halfword destination is address 1. Whether that access succeeds is platform-dependent.

10. **Terminal nonreturn:** If the zero-address call faults or never returns, the routine does not reach its declared return. Earlier effects are not undone by any visible mechanism.

## Uncertainties

The following remain **documented_only**:

- The actual calling convention and completeness of the recovered signature.
- The identities and contracts of all dependencies.
- Whether secondary-register values are intentional outputs.
- Whether the represented region has a valid independent entry boundary.
- Whether low numerical addresses denote ordinary memory, special mappings, or artifacts of an inaccurate static interpretation.
- Exact global aliasing and the source address of the upper-24-bit fragment.
- Endianness, alignment rules, and fault handling.
- Exact signed-overflow and signed-left-shift behavior.
- Whether the final zero-address target is executable.
- Whether any dependency returns normally or modifies hidden state needed later.
- Whether the routine has any meaningful application-level success condition.

These uncertainties must not be resolved by inventing safe defaults, ownership rules, or inferred helper semantics.

## Evidence

This contract is based only on the supplied static function representation and its warning about overlapping global symbols.

Directly supported observations include the declared widths, explicit argument forwarding, straight-line control structure, ordered memory accesses, masks and shifts, global writes, and final zero-address call.

The zero result of the repeated 23-bit right shift and the stated address ranges are arithmetic deductions from the represented 32-bit unsigned values.

Application purpose, dependency semantics, runtime validity, and successful completion are not established. No implementation access, runtime execution, or tests were used.
