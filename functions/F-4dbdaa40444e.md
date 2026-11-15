<!-- Trusted provenance: F-4dbdaa40444e; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-4dbdaa40444e

## Purpose

**Established behavior:** F-4dbdaa40444e is a nonterminating, stateful sequence of indirect calls, byte accesses, and 32-bit word accesses. It repeatedly derives addresses and values from dependency results, incoming machine state, and memory.

No application-level purpose is established. In particular, the supplied representation does not justify describing this function as an allocator, serializer, device initializer, or collection operation. Repeated offsets and scaled addresses suggest structured state manipulation, but that interpretation is **inferred intent**, not a supported interface guarantee.

This contract describes the visible operations and their ordering. Dependency behavior, executable validity, and the interpretation of the recovered control flow remain **documented_only** where evidence is insufficient.

## Interface

### Explicit inputs

The visible interface has four 32-bit arguments:

| Argument position | Established use |
|---|---|
| First | No explicit use in the supplied body. |
| Second | Passed to the first indirect call of an iteration. Replaced before the next iteration by a dependency result shifted left two bits. |
| Third | No explicit use in the supplied body. |
| Fourth | Its low eight bits supply two early byte writes. Later replaced internally by zero, so subsequent iterations use zero for those writes. |

Lack of explicit use does not establish that the first and third arguments are unobservable to machine-level callees. The recovered call signatures are incomplete.

### Additional machine-state inputs

The body also consumes state not represented by the four arguments:

- A 32-bit base used for initial byte accesses.
- A 32-bit value shifted left 28 bits for an early word write.
- An unsigned 32-bit value shifted right 22 bits to form an early read address.
- A link-related value used to select the first indirect call target.
- An additional register value passed as a second argument at one later call.
- A global 32-bit storage value read during each iteration.

These are required parts of the effective input state. Their initialization, validity, and calling convention are not established.

### Outputs and termination

There is no visible normal return or explicit returned value. The body repeats unconditionally.

Observable outputs are:

- Memory writes.
- Indirect calls and their arguments.
- Any additional effects of those calls.
- Any fault, trap, suspension, or nonreturn caused by a call or memory access.

There is no visible error code, bounds check, retry policy, rollback, or cleanup path.

### Arithmetic and access notation

For the behavioral descriptions below:

- A **byte** is eight bits.
- A **word** is 32 bits.
- A **scaled result** means a dependency’s 32-bit result shifted left two bits.
- A **byte-selected address** is formed by reading an unsigned byte from a specified address and adding 17 to that byte’s value.
- A **word slot with scale** is the supplied result multiplied by that scale, then increased by 20.

These definitions describe address expressions, not validated objects or arrays.

Unsigned shifts have their ordinary logical meaning. Byte stores retain only the low eight bits. For signed-looking shifts and multiplication, a 32-bit modular interpretation is useful for describing expected machine behavior, but overflow and address-wrap behavior require machine-level confirmation. This is not a guarantee of portable signed-arithmetic semantics.

## Behavior

### Ordering rules

The following stages occur in the stated order within each iteration.

Indirect calls separate many of the accesses. They are not pure computations unless independently established. A later memory read must therefore use memory as it exists at that point, including possible changes made by dependencies.

A saved byte or word value remains the value previously obtained; it is not implicitly refreshed. This distinction matters where one byte selects both a byte write and a later word write across intervening calls.

The descriptions below identify result roles by their use, not by source-local names.

### 1. Initial call sequence and base-byte initialization

The iteration begins by retaining the low eight bits of the current fourth argument.

The function then:

- Calls a target selected using the incoming link-related state, passing the current second argument.
- Obtains a 32-bit result from a subsequent indirect call.
- Passes that result to another indirect call.
- Makes four further indirect calls with no explicit arguments shown, retaining the final result.

It then writes the retained argument byte:

1. To the current base plus 17.
2. To the current base plus 1.

The order is significant if these locations have observable access semantics or dependencies later examine them.

The retained final dependency result, shifted left two bits, is passed to another indirect call. Four further calls follow; the final result selects the first scaled word destination.

### 2. Initial word write and independent word read

The function writes a word at:

- Destination: 16 times the newly obtained result, plus 20.
- Value: the incoming shift-source value shifted left 28 bits.

It next reads a word from:

- Address: the incoming unsigned selector shifted right 22 bits, plus 20.

This is a direct address calculation. The supplied representation does not show an additional base pointer.

The first result multiplied by four is then passed to a dependency. A subsequent call produces a word retained for a later write.

The previously read word, shifted right 22 bits and then left 28 bits, is passed to another call. Four further calls produce a result used for a word destination.

The function writes zero to that result’s word slot with scale four.

### 3. First byte-selected write sequence

After another argument-bearing call and four calls without explicit arguments, a new 32-bit result is obtained.

The function:

1. Reads the byte at the current base plus 1.
2. Writes the low byte of the new result shifted left two bits to the corresponding byte-selected address.

It then passes the shifted result to a dependency. A later call produces a word, which is passed to another call together with the earlier result shifted left 30 bits.

Four further calls produce a result used for another call argument.

Before that next call, the function:

1. Reads the byte at the current base plus 1 again.
2. Saves this byte.
3. Writes the low byte of the earlier result, masked to 30 bits and shifted right 20 bits, to the saved byte’s selected address.

The fresh read is required: dependencies may have changed the selector since the preceding byte write.

The new dependency result shifted left two bits is passed onward, followed by four calls. The function then writes a word at the **saved byte value plus 20**, using the earlier retained word shifted left two bits.

The selector for this word write is not reread after those intervening calls.

### 4. Global-derived byte update and clearing

The function reads the global 32-bit storage value once and retains it.

After a call using an earlier scaled result and four further calls, a 64-bit result is obtained. Its low 32-bit half, shifted left two bits, is retained as an address-driving value.

The function then:

- Reads the byte at the current base plus 1.
- Writes the low byte of the global snapshot shifted right 22 bits to the selected address.

The high 32-bit half of the 64-bit result is passed to a dependency. A later 32-bit result is passed to another dependency together with the retained low-half-derived value shifted left 28 bits.

After four further calls, the function rereads the byte at the current base plus 1 and writes zero to its selected address.

The most recent result shifted left two bits is passed onward. Four more calls produce a result whose shifted value is passed twice, as two explicit arguments, to another call.

### 5. First pair of related word writes

Four calls after that two-argument call produce a new result. Its scaled result becomes the value for two word writes.

The first write is:

- Destination: the address-driving value from the low half in stage 4, multiplied by 4096, plus 20.
- Value: the new scaled result.

A dependency is called with the address-driving value shifted left 14 bits. Four further calls produce a destination selector.

The second write is:

- Destination: that selector’s word slot with scale four.
- Value: the same retained scaled result.

The first write precedes all calls leading to the second. The two writes are not established to be adjacent, disjoint, or atomic.

### 6. Derived-base byte writes

After another argument-bearing call and four further calls, another 64-bit result is obtained. Its low half shifted left two bits becomes the next address-driving value.

The function:

1. Reads a byte from the stage-4 address-driving value multiplied by 16,384, plus 1.
2. Writes to that byte’s selected address the low byte of the retained stage-5 word value, masked to 30 bits and shifted right 20 bits.

The high half of the new 64-bit result is passed to a dependency. A following result is passed to another dependency.

A further call produces a 64-bit result whose low and high halves are passed, in that order, as two explicit arguments to another call. Two calls follow, the latter supplying a result used later.

The function then reads a byte at the stage-6 address-driving value multiplied by 268,435,456, plus 1, and writes zero to the selected address.

After a call using the latest result shifted left two bits and four further calls, a new result is retained. The function also establishes a selector base equal to four times the stage-6 address-driving value.

It reads a byte at that selector base plus 1 and writes zero to the selected address.

### 7. Repeated clearing and second pair of word writes

A dependency receives the stage-6 address-driving value shifted left ten bits. A subsequent result is passed to another dependency together with the word retained in stage 2 shifted left 28 bits.

After four more calls, the function again:

- Reads a byte at the selector base plus 1.
- Writes zero to the selected address.

This second clearing is a distinct access. It cannot be removed merely because the preceding write also stored zero: the selector or destination may have changed during the intervening calls.

After another argument-bearing call and four further calls, the function performs a word write:

- Destination: the scaled result retained in stage 6, multiplied by 4096, plus 20.
- Value: the corresponding unscaled result shifted left 14 bits.

It then calls a dependency with the scaled result shifted left 14 bits. Another call follows. A subsequent call receives:

- The just-written value shifted left 28 bits.
- An additional register-derived value whose origin is not established.

Four further calls produce another destination selector. The function writes the same retained word value to this selector’s word slot with scale four.

Under a 32-bit modular interpretation, the first argument formed by shifting a value left 14 bits and then left 28 bits is zero. This simplification does not establish anything about the unknown second argument.

### 8. Another byte update and clearing

Following another call and four further calls, a new result is obtained and scaled.

The function reads a byte at the selector base plus 1 and writes to its selected address the low byte of the stage-7 word value, masked to 30 bits and shifted right 20 bits.

A dependency then receives the earlier stage-6 result shifted left 26 bits. A following result is passed to another dependency together with the newly scaled result shifted left 28 bits.

A subsequent 64-bit result is split into low and high halves and passed as two explicit arguments to another dependency. Two further calls follow.

The function rereads the byte at the selector base plus 1 and writes zero to the selected address.

### 9. Third pair of related word writes

After a call using the latest result shifted left two bits and four further calls, another scaled result becomes a retained word value.

The function writes it to:

- The stage-8 scaled result multiplied by 4096, plus 20.

A following dependency receives four explicit arguments:

1. The stage-8 scaled result multiplied by 1024.
2. The same value again.
3. The corresponding unscaled result shifted left 14 bits.
4. The retained word value.

Four further calls produce a selector. The function writes the same retained word value to that selector’s word slot with scale four.

### 10. Final word writes and next-iteration state

Another call and four further calls produce a new result. The function writes that result multiplied by four to its own word slot with scale 16.

The current fourth argument is then set to zero.

A dependency receives the scaled result three times. Four further calls produce a selector, and the previous scaled result is written to its word slot with scale four.

Another call and four further calls produce a result used in two ways:

- Its value multiplied by four is written to its word slot with scale 16.
- Its value multiplied by 1,073,741,824 becomes the base for the next iteration.

A dependency receives this latest scaled result three times. Four further calls produce another selector. The function writes the retained scaled result to that selector’s word slot with scale four.

After a final call and four further calls:

- The last dependency result shifted left two bits becomes the next second argument.
- The preceding word-slot selector shifted left three bits becomes the next shift-source value.
- Zero is written first to the new base plus 17, then to the new base plus 1.
- The next second argument also becomes the next unsigned selector.
- The link-related state used by the next iteration’s first call is replaced by a fixed internal continuation value.

The iteration repeats without a termination test.

## Side effects

### Memory layout and aliasing

Visible accesses use offsets 1, 17, and 20, but no structure definition is established.

Byte-selected destinations lie numerically between 17 and 272 inclusive before any machine-specific address interpretation. They are not offsets from the current base.

Word destinations use scales including four, 16, and 4096. Some may be unaligned. No alignment correction is visible.

Any locations may alias each other, global storage, or memory accessed by dependencies. Consequently:

- A byte write can affect a later word read or write.
- A word write can alter a later selector byte.
- Repeated address expressions do not imply repeated effective destinations when a selector is reread.
- Intervening calls prevent assuming stable memory.

Endianness is not specified, so overlapping byte/word effects cannot be resolved further.

### Ownership and failure

No visible allocation, release, ownership transfer, or lifetime management is established. Passing a value to a dependency does not establish that it is a pointer or that the dependency owns it.

All selected call targets and accessed addresses must be valid for execution to reach the next operation. Violations have no visible handling. Earlier effects remain observable if execution faults later; no transactional behavior is shown.

## Dependencies

All visible calls are indirect. Their semantic identities and contracts are unavailable.

The only supplied opaque function identity is **F-4dbdaa40444e**. No dependency identity can be assigned without inventing provenance; unresolved dependency identities are therefore **documented_only**.

The body visibly uses dependencies with:

- Zero through four explicit arguments.
- 32-bit results.
- 64-bit results subsequently split into low and high halves.
- An additional register-derived argument not tied to an established explicit result.

Calls shown without explicit arguments are not proven to have no machine-level inputs. No dependency is assumed to preserve globals, avoid memory writes, return normally, or manage memory.

## Acceptance examples

These are conditional static acceptance examples, not executed tests.

1. **Initial byte truncation.** If the fourth argument is `0x123456AB` and execution reaches stage 1, the two early stores write `0xAB`, first at base plus 17 and then at base plus 1.

2. **Unsigned selector interpretation.** If a selector-byte read yields `0xFE`, its selected byte-write address is 271, not base plus 271 and not a sign-extended negative address.

3. **Fresh versus saved selectors.** If the two stage-3 selector reads yield 3 and 9, their byte destinations are 20 and 26. The later stage-3 word destination is 29, even if a dependency subsequently changes the selector byte.

4. **Byte truncation after shifting.** If the stage-3 result is `0x0000007F`, the first selected byte store writes `0xFC`.

5. **Masked extraction.** If a masked-extraction source is `0x3FF00000`, shifting its low 30 bits right 20 produces `0x3FF`; the byte store retains `0xFF`.

6. **64-bit result separation.** If a 64-bit result is `0x1122334455667788`, its low half is `0x55667788` and its high half is `0x11223344`. Their argument order must not be reversed.

7. **Repeated clearing remains observable.** If a dependency changes the selector between the two stage-7 clearing operations, each operation uses its own read and may clear a different address.

8. **Subsequent-iteration initialization.** Once execution reaches stage 10 and begins another iteration, the fourth-argument-derived byte is zero. The initial caller-supplied low byte is not reused.

9. **No normal completion.** If every call returns and every access succeeds, completion of stage 10 leads to another iteration, not a return.

## Uncertainties

The following remain **documented_only**:

- Application purpose and object interpretation.
- Initialization and ABI status of additional machine-state inputs.
- Whether the recovered indirect targets and control-flow boundaries faithfully represent executable instructions.
- Dependency identities, complete signatures, side effects, and nonreturn behavior.
- Endianness, alignment rules, memory mapping, and fault semantics.
- Exact signed-overflow and address-wrap behavior.
- Whether any selected address refers to ordinary RAM, device memory, or invalid space.
- Whether the apparently unused explicit arguments are observable through an incomplete recovered calling convention.
- Whether overlapping global storage affects other named state.

The supplied representation also reports omitted unreachable material and overlapping global symbols. Those annotations do not establish the behavior of omitted material or a reliable global layout.

## Evidence

This contract is based solely on static inspection of the supplied representation.

Established evidence includes the visible access widths, argument expressions, indirect-call order, selector rereads, retained values, global read, state replacements, and unconditional repetition.

The interpretation of repeated accesses as structured state manipulation is inference only. Modular arithmetic conclusions are conditional on a 32-bit machine interpretation where the recovered signed-looking operations require confirmation.

No runtime tests, executable inspection, dependency inspection, or implementation validation have been performed.
