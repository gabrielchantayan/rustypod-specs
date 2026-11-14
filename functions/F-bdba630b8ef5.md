<!-- Trusted provenance: F-bdba630b8ef5; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-bdba630b8ef5

## Purpose

F-bdba630b8ef5 performs an ordered sequence of indirect calls, 32-bit memory transfers, bit-field extraction, byte updates, and final state clearing. Its visible behavior is primarily mutation of externally supplied or externally derived memory.

The supplied static representation does not establish a higher-level purpose. In particular, it does not establish that the function initializes an object, implements a supervisor operation, manages a collection, or releases resources. Such interpretations would require additional evidence.

This contract describes the behavior visible in the supplied representation, subject to its unresolved calling convention and arithmetic semantics. It does not establish that the represented entry point can execute successfully in an ordinary process: the sequence includes accesses to very low addresses and ends with an indirect call to address zero.

## Interface

### Explicit inputs and output

The displayed interface has two explicit inputs:

- A 32-bit value whose bits are not explicitly consumed by the visible body.
- A signed 32-bit value used immediately to compute a memory destination and subsequently passed to an indirect dependency.

There is no explicit return value. A normal return is represented after the final indirect call, but normal completion depends on that call returning.

The first input is not established as irrelevant to execution. The displayed calling convention may omit register-level dependencies, and the first indirect call may observe state not described by its displayed argument list.

### Additional incoming state

The body also depends on incoming machine state not accounted for by the two explicit parameters:

- A value used as the base for an initial 32-bit read at offset 20.
- A 32-bit value used in the first two word stores.
- A value used to derive the first indirect-call target.
- Secondary results or residual register values from several indirect calls.
- Two named global storage locations and other low-address memory.

These are required environmental inputs, not values that an implementation may safely replace with zero.

For discussion, the two named global locations are called **global A** and **global B**. These are contract labels, not assertions about their meaning. Global A is read on two separate occasions. Global B receives two explicit writes.

### Widths and notation

The supplied representation distinguishes:

- 8-bit byte loads and stores.
- 16-bit stores.
- 32-bit values, loads, and stores.
- 64-bit dependency results split into low and high 32-bit halves.

Offsets and shift counts below are decimal. “Low byte” means the least significant eight bits of a value, not necessarily the first byte in its memory representation.

For compactness:

- **Word slot of an index** means the address obtained by multiplying that index by four and adding 20.
- **Byte-selected destination** means: load one unsigned byte at the specified selector address, add 17 to the loaded value, and use that sum as the destination address.

These are address calculations, not validated object or array accesses.

Unsigned right shifts discard low bits and introduce zeros. Byte stores retain only the low eight bits. Address wrapping, signed multiplication overflow, and signed left-shift behavior are not fully established by the supplied representation; they must not be silently converted into checked arithmetic or assumed to follow portable language rules.

## Behavior

### Execution conditions

There are no visible conditional branches, loops, bounds checks, null checks, retries, or explicit error-handling paths. The following sequence describes the path taken if each preceding access and call completes and returns normally.

Dependency calls occur between many of the explicit accesses. They may change shared memory. Consequently, the accesses below are ordered events, not a collection of assignments that may be freely reordered.

### 1. Initial stores and captured fields

1. Store the incoming auxiliary 32-bit value in the word slot selected by the second explicit input.
2. Read a 32-bit unsigned word at the incoming base plus 20. Retain this value locally across subsequent calls.
3. Invoke an indirect dependency with the second explicit input.
4. Execute an indirect-call chain. A 32-bit result is forwarded to the next call; later in the chain, another result supplies an index.
5. Store the same incoming auxiliary value in the word slot selected by that returned index.
6. Read global A as a 32-bit value. Retain this particular read independently of later changes to global A.
7. Continue the call chain, beginning with a call receiving the returned index multiplied by four.
8. Shift the initially captured word right by 22 and shift the captured global A value right by 23.
9. Store the shifted initial word in a word slot selected by a secondary register result from the intervening calls.
10. Invoke a dependency with four displayed arguments: that selector twice, the shifted global A value, and the shifted initial word.

The shifted initial word is at most 1,023. The shifted global A value is at most 511.

### 2. Repeated word propagation and first byte updates

After another indirect-call chain:

1. Store the shifted initial word in a word slot selected by a returned index.
2. Invoke a dependency with four times that index, then continue the call chain.
3. Store a 32-bit zero in a word slot selected by a later secondary unsigned result.
4. Invoke a dependency with that secondary result, then continue the call chain.
5. Store a 32-bit zero in a further returned-index word slot.

The fifth store is zero because its displayed value is the already right-shifted global A value shifted right by another 23 bits. A 32-bit value shifted right by 23 twice is necessarily zero.

Immediately after that store, read a 32-bit word at:

- The secondary unsigned result masked to its low 29 bits;
- Then shifted right by 19;
- Then increased by 20.

The offset before adding 20 is between zero and 1,023. It need not be word-aligned.

After further calls, shift this captured word right by 22. Store its low byte at the byte-selected destination whose selector address is one.

The selector address is one because the earlier doubly shifted global A value is zero. This does **not** imply the destination address is fixed: its destination depends on the byte currently stored at address one.

Next:

- Invoke a dependency with the same secondary register value repeated three times and a fourth argument of zero.
- Execute another call chain, including a call with a displayed zero argument.
- Read a selector byte at four times the captured shifted word plus one.
- Store zero at the corresponding byte-selected destination.
- Invoke the next dependency with a returned index shifted left by two.

The selector address in this second byte update lies between one and 4,093, before any target-specific address effects.

### 3. Low-address clearing and first global B update

The next call sequence includes:

- A call receiving the same returned index shifted left by two in both displayed argument positions.
- A call producing a 32-bit value and another call receiving that value.
- A byte read at address one followed by a zero store at the resulting byte-selected destination.

A secondary result is then retained for later use. A dependency receives this value three times.

Subsequent calls forward a 32-bit result together with the earlier call-produced value shifted right by 23. A later returned index is used as an argument, but the next explicit zero store does not use it: instead, the function rereads the byte at address one and clears that byte-selected destination.

This second selector byte is also retained locally as an unsigned base.

After further calls, a dependency produces a new 32-bit value. The function:

1. Writes that value to global B.
2. Passes the value to another dependency.
3. Invokes another dependency.
4. Reads a 32-bit word at the retained byte-derived base plus 116.
5. Writes that word’s low byte at the same retained base plus one.

The retained base is not reread from address one for this last transfer. An intervening dependency changing address one therefore does not change this particular base.

### 4. Word-to-byte transfers and indexed propagation

A dependency is invoked with the saved secondary result shifted left by 11. A later dependency returns a 64-bit result. Its low half, high half, and the saved global B value shifted right by 23 are passed onward as three displayed arguments.

After another call chain:

1. Obtain an index and multiply it by four to form a base.
2. Read a word at that base plus 116.
3. Write the word’s low byte at that base plus one.
4. Invoke dependencies, beginning with that base as an argument.
5. Obtain another index and form a second base by shifting it left by two.
6. Repeat the word-read/byte-write operation using the **first** base.

The second transfer is a fresh read from the first base plus 116, not a reuse of the previous word. Intervening calls can make the two written bytes differ.

Next, invoke a dependency with:

- The saved secondary result shifted left by 12.
- The second base twice.
- The second index shifted left by 18.

Further calls forward results. A later returned index supplies a new four-times-index base. At this base, the function reads a 32-bit word at offset 116 and writes its low byte at offset one. Retain the entire word for later stores and selector calculations.

After further calls:

- Store the retained full word at an address formed from a returned index multiplied by 16, plus 20.
- Invoke a dependency with the second base shifted right by two, then left by 16.
- Continue the result-forwarding call chain.
- Store the retained word in another returned-index word slot.
- Continue calls, beginning with four times that index.
- Obtain another index, shift it left by two, and store that result at the second base with its low two bits cleared, plus 20.
- Invoke a dependency with the stored shifted result in both displayed argument positions.

Because the second base was formed by shifting left by two, its low two bits are already zero under a 32-bit bit-vector interpretation. That observation does not authorize changing the surrounding dependency calls or their order.

### 5. Further word stores and second global A snapshot

Another call sequence forwards results, then:

1. Stores a 32-bit zero in a returned-index word slot.
2. Continues calls beginning with four times that index.
3. Receives a 64-bit result.
4. Shifts its low 32-bit half left by two.
5. Stores the shifted value in the word slot selected by its high 32-bit half.
6. Reads global A again.
7. Invokes a dependency with the shifted value, the high half, and the shifted value twice more.

This second global A read is not equivalent to the earlier read. Intervening dependencies and explicit writes may have changed its value.

After further calls, store the same shifted value in another returned-index word slot. Then read a 32-bit word at the newly captured global A value plus 20.

Continue the call chain, beginning with four times the latest returned index.

### 6. Final byte writes, global reconstruction, and termination

A subsequent dependency returns another 64-bit result. Retain its low and high halves separately.

Before the next call:

- Shift the full word retained in phase 4 right by 23 and add one.
- Read a byte at that selector address.
- Store the low byte of the word read in phase 5 at the resulting byte-selected destination.

Invoke a dependency with the latest high half repeated three times and the latest low half shifted left by two as its fourth argument.

After another call chain:

- Mask the retained low half to its low 30 bits.
- Shift it right by 21 and add one to obtain a selector address.
- Read a byte there.
- At the resulting byte-selected destination, store the low byte of the phase-5 word shifted right by 22.
- Invoke the next dependency with a newly returned index shifted left by two.

Further calls include another call with the same shifted index in both displayed argument positions, followed by calls that obtain and forward a 32-bit result.

The final visible actions are:

1. Obtain an 8-bit result from an indirect dependency.
2. Store zero through a secondary pointer result associated with that call sequence.
3. Reconstruct and write global B using a three-byte global fragment as its upper 24 bits and the returned byte as its lower eight bits.
4. Store a 16-bit zero at the address obtained by masking the retained low half to 30 bits and shifting right by seven.
5. Invoke address zero with three displayed zero arguments.
6. Return only if that invocation returns.

The source of the upper 24 bits of the final global B value is evaluated after the pointer-directed byte clear. Possible aliasing must therefore be preserved.

## Side effects

The function explicitly performs word, byte, and halfword stores; reads shared and low-address memory; writes global B twice; and invokes numerous indirect dependencies.

Global A is explicitly read twice but has no explicit direct write. This is not a preservation guarantee: dependencies and aliases may modify it.

Byte-selected destinations range from address 17 through address 272 because their selectors are unsigned bytes. Their selector addresses have different calculations and must not be confused with the final destinations.

There is no visible allocation, deallocation, reference-count operation, lock operation, or ownership transfer. None may be inferred from the indirect calls. Memory passed to dependencies or accessed directly must remain valid for the actual execution, but the supplied evidence does not identify who owns it or whether dependencies retain or release it.

There is no rollback. A fault or non-returning dependency can leave any already completed prefix of the side effects in place.

## Dependencies

No resolved dependency identity is supplied beyond F-bdba630b8ef5 itself. The following opaque IDs label dependency **roles**, not proven distinct function identities:

| Opaque ID | Visible dependency role |
|---|---|
| F-000000000001 | Initial indirect call, with a target derived from incoming machine state and the second explicit input as a displayed argument. |
| F-000000000002 | Repeated indirect call sites displaying no explicit arguments and producing either ignored or subsequently consumed results. |
| F-000000000003 | Indirect call sites forwarding 32-bit values, indices, shifted indices, or repeated arguments. |
| F-000000000004 | Indirect call sites producing results represented as 64-bit values. |
| F-000000000005 | Final indirect call producing the byte used in the last global B write. |
| F-000000000006 | Final call to address zero with three displayed zero arguments. |

These labels do not authorize merging calls. Distinct call sites have distinct displayed target derivations, and every intervening call remains potentially observable.

No dependency is established to preserve globals, preserve pointed-to memory, terminate, report errors conventionally, or own/release memory. Displayed argument lists may also be incomplete descriptions of machine-level inputs.

## Acceptance examples

These examples are conditional static consequences, not executed tests.

1. **Initial field extraction.** If the first captured word is `0xFFFFFFFF`, the value stored after shifting it right by 22 is 1,023. If the first global A snapshot is also `0xFFFFFFFF`, its first extracted value is 511, and shifting that extracted value right by 23 again produces zero.

2. **Full word versus low byte.** If a phase-4 read at a base plus 116 yields `0x123456AB`, the byte write at that base plus one writes `0xAB`. Later stores of the retained full word write `0x123456AB`, not merely `0xAB`.

3. **Fresh repeated reads.** If a dependency changes the word at the first phase-4 base plus 116 between the two transfers, the second byte store reflects the changed word. Reusing the first read would not satisfy the visible sequence.

4. **Byte-selected destination.** If a selector read yields 200, the corresponding destination is address 217. If the stored value before truncation is 1,023, the byte written there is 255.

5. **Separate global snapshots.** If dependencies change global A after its first read, the final phase must use its second captured value where specified. Caching the first snapshot for both phases changes behavior.

6. **Final global construction.** If the final upper three-byte fragment is `0x123456` and the returned byte is `0xAB`, the constructed 32-bit value is `0x123456AB`. The fragment must reflect state after the preceding pointer-directed zero store.

7. **Failure prefix.** If a dependency never returns after the first global B write, later stores and the final call do not occur. The earlier global B write is not undone.

## Uncertainties

The following are **documented_only**:

- The intended application-level purpose and meanings of the global locations.
- The complete machine-level calling convention and validity of the displayed function boundary.
- The origin and exact association of secondary register results with individual calls.
- Concrete dependency identities, complete argument lists, and dependency side effects.
- Target endianness, signed-overflow behavior, address wraparound, and alignment requirements.
- Whether low-address accesses address ordinary memory, device state, a mapped runtime structure, or an invalid region.
- Whether address zero is a callable entry, a deliberate trap, or an artifact of the supplied representation.
- Aliasing among globals, computed destinations, and dependency-accessible memory.
- Exception behavior, error reporting, ownership, concurrency requirements, and successful normal completion.

An inferred bit-vector interpretation explains several shifts and masks, but does not establish all target arithmetic rules.

## Evidence

Established static evidence consists solely of the supplied body: its displayed input widths, indirect calls, memory-access widths, arithmetic expressions, result forwarding, explicit store order, and absence of visible branches.

The range bounds, guaranteed zero after two successive unsigned right shifts by 23, and low-byte truncations are mathematical consequences of the displayed widths and operations.

Application intent, dependency semantics, and executable validity are not established. No runtime tests have been performed.
