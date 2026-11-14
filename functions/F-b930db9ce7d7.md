<!-- Trusted provenance: F-b930db9ce7d7; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-b930db9ce7d7

## Purpose

The supplied static representation describes a non-returning operation that repeatedly invokes unresolved dependencies, computes addresses from 32-bit values, and performs byte and 32-bit memory accesses.

Its application-level purpose is not established. The repeated accesses at offsets 1, 17, and 20 may suggest a structured state representation, but the evidence does not establish an object model, device interface, allocator, or protocol. These offsets must not be assigned application meanings without additional evidence.

The contract below records the visible access and data-flow sequence. It does not assume that dependency calls preserve memory, global state, or resource ownership.

## Interface

### Explicit inputs

Four incoming argument slots are shown:

| Argument | Visible width | Established use |
|---|---:|---|
| First | 32 bits | No explicit use in the supplied body. |
| Second | 32 bits | No explicit use in the supplied body. |
| Third | 32 bits | Initially contributes to a dependency argument shifted left by 28 bits. Later replaced by a dependency result and used in a memory write. |
| Fourth | 32 bits | Initially determines an address scaled by 4096 and a dependency argument shifted left by 14 bits. Replaced near the end of each iteration. |

Absence of an explicit use does not establish that the first two argument slots are irrelevant to the actual machine-level execution. The recovered call interfaces are incomplete.

### Additional incoming context

Execution also depends on:

- An incoming continuation-like value used to form the first indirect call target.
- An incoming register-like value used to address a byte during the first iteration.
- An unnamed 32-bit global location read late in every iteration.
- Additional call-result register state used as one argument without a separately established defining value.

The ordinary four-argument interface is therefore insufficient to describe all prerequisites.

### Outputs and completion

No normal return is visible. The body ends in an unconditional repetition.

There is no established return value, status result, error object, or completion notification. A dependency that does not return, a memory-access fault, or an invalid indirect transfer can prevent further progress; no local recovery is shown.

### Width and address conventions

The visible representation uses:

- 8-bit byte loads and stores.
- 32-bit word loads, stores, arguments, and address computations.
- Selected dependency results represented as 64-bit pairs, split into low and high 32-bit halves.
- Zero extension when a loaded byte becomes an address component.
- Low-byte truncation on byte stores.

The arithmetic descriptions below use a **32-bit machine interpretation**, with multiplication and left shift retaining the low 32 bits. This is the natural interpretation of the recovered widths, but overflow behavior and the target address model remain documented-only rather than independently verified.

For compactness:

- **wrap** means retain the low 32 bits.
- **low byte** means retain bits 0–7.
- **field extraction** means mask with `0x3fffffff`, then logically shift right by 20.

A byte selected through another byte is a two-access operation: first load an unsigned byte, then access address 17 plus that byte. The second address is consequently between 17 and 272 inclusive. It is not the original base plus 17.

No alignment, bounds, null, or access-permission checks are visible.

## Behavior

### Dependency notation

All indirect calls are described under the opaque dependency designation **F-UNRESOLVED**. This designation is an unresolved dependency class, not a claim that all call sites target the same function.

Calls remain ordered as shown. A call without an explicit argument in this contract has no argument established by the supplied representation; it must not be assumed to receive no machine-level inputs or have no side effects.

The descriptive names below identify saved values for this contract, not source identifiers.

### 1. Establish and distribute the first word

At the start of an iteration, four indirect calls occur. The fourth produces a 32-bit result.

The operation:

1. Forms the **first word** by shifting that result left by two bits.
2. Stores the first word as a 32-bit quantity at address wrap(fourth argument × 4096 + 20).
3. Calls a dependency with wrap(fourth argument shifted left by 14).
4. Makes four further calls, the last producing a 32-bit **first destination index**.
5. Stores the same saved first word at wrap(first destination index × 4 + 20).
6. Calls a dependency with wrap(first destination index × 4).

The two stores need not target different locations. No reread of the first stored word is used to supply the second store.

### 2. Publish an extracted byte and consume a paired result

Four more calls occur. The fourth produces a paired result with low and high 32-bit halves.

The operation saves the low half shifted left by two as the **first scaled value**. It then:

1. Loads a byte at wrap(incoming register-like value × 4096 + 1).
2. Stores the low byte of the first word’s field extraction at address 17 plus that loaded byte.
3. Calls a dependency with the paired result’s high half.
4. Calls another dependency to obtain a 32-bit result, then passes that result to the next dependency.
5. Obtains another paired result and passes its low and high halves to the following dependency.
6. Makes two further calls, the last producing a 32-bit result.

It next loads a byte at wrap(first scaled value × `0x10000000` + 1), stores zero at address 17 plus that byte, and calls a dependency with the most recent 32-bit result shifted left by two.

The first scaled value is already a multiple of four. Under wrapping arithmetic, its multiplication by `0x10000000` therefore selects one of four high-address bases.

### 3. Perform two independently selected byte clears

Four calls follow, the last producing a **second index**. A **persistent base** is formed as wrap(first scaled value × 4).

The operation then:

1. Loads a byte at wrap(persistent base + 1).
2. Writes zero at address 17 plus that byte.
3. Calls a dependency with wrap(first scaled value shifted left by 10).
4. Obtains a 32-bit result and passes it, together with wrap(third argument shifted left by 28), to another dependency.
5. Makes four further calls, the last producing another 32-bit result.
6. Reloads the byte at wrap(persistent base + 1).
7. Writes zero at address 17 plus the newly loaded byte.
8. Calls a dependency with the latest result shifted left by two.

These clears use separate selector reads. They are not necessarily writes to the same address.

### 4. Distribute the second word

Four calls occur. The operation then forms the **second word** as wrap(second index shifted left by 14), and:

1. Stores it at wrap((second index × 4) × 4096 + 20).
2. Calls a dependency with wrap((second index × 4) shifted left by 14).
3. Makes another call.
4. Calls a dependency with the second word shifted left by 28 and an additional register-derived value.

Under 32-bit wrapping arithmetic, the first argument in step 4 is always zero: the second word already has 14 zero low bits.

Four more calls follow, the last producing a destination index. The operation stores the saved second word at wrap(destination index × 4 + 20), then calls a dependency with wrap(destination index × 4).

### 5. Publish the second extracted byte and clear again

Four calls occur, the last producing a **third index**. The operation:

1. Reloads the byte at wrap(persistent base + 1).
2. Stores the low byte of the second word’s field extraction at address 17 plus that byte.
3. Calls a dependency with wrap(second index shifted left by 26).
4. Obtains a 32-bit result and passes it to another dependency, together with wrap((third index × 4) shifted left by 28).
5. Obtains a paired result and passes its low and high halves onward.
6. Makes two more calls, the last producing a 32-bit result.
7. Reloads the byte at wrap(persistent base + 1).
8. Writes zero at address 17 plus that byte.
9. Calls a dependency with the latest result shifted left by two.

The extraction in step 2 depends on bits 6–13 of the second index after final byte truncation. It is not a high-byte extraction from an unshifted index.

### 6. Distribute a third word

Four calls follow, the last producing a value from which the **third word** is formed by shifting left by two.

The operation stores that word at wrap((third index × 4) × 4096 + 20), then calls a dependency with four explicit arguments:

- wrap((third index × 4) × 1024);
- the same value again;
- wrap(third index shifted left by 14);
- the third word.

Four further calls produce a destination index. The third word is stored at wrap(destination index × 4 + 20), followed by a call receiving wrap(destination index × 4).

### 7. Perform two scaled-word rounds

Four calls produce a **fourth index**. The operation stores wrap(fourth index × 4) at wrap(fourth index × 16 + 20), then calls a dependency with that scaled value repeated three times.

Four more calls produce another destination index. The same saved scaled value is stored at wrap(destination index × 4 + 20), followed by a call receiving wrap(destination index × 4).

The pattern repeats with a newly returned **fifth index**:

1. Store wrap(fifth index × 4) at wrap(fifth index × 16 + 20).
2. Save a **terminal base** equal to wrap(fifth index × `0x40000000`).
3. Call a dependency with wrap(fifth index × 4) repeated three times.
4. Make four calls to obtain a destination index.
5. Store the saved scaled value at wrap(destination index × 4 + 20).
6. Call a dependency with wrap(destination index × 4).

The terminal base is one of 0, `0x40000000`, `0x80000000`, or `0xc0000000`.

### 8. Clear two terminal bytes twice

Four calls produce a **selector word**. Before the next call, the operation writes zero, in order, to:

1. wrap(terminal base + 17);
2. wrap(terminal base + 1).

It then calls a dependency with the selector word shifted left by two, obtains another result, and passes that result to another dependency.

Four further calls produce a 32-bit result. The same two direct zero stores are repeated in the same order, followed by a dependency call receiving the latest result shifted left by two.

These are direct stores relative to the terminal base, not selector-mediated stores.

### 9. Read a word through an extracted address

Four calls produce a **sixth index**. The operation:

1. Stores wrap(the destination index retained from the fifth-index round shifted left by 31) at wrap(sixth index × 16 + 20).
2. Reads a 32-bit **loaded word** from address 20 plus the selector word’s field extraction.
3. Calls a dependency with wrap(sixth index × 4).
4. Obtains a new value for the third argument.
5. Calls a dependency with wrap((loaded word logically shifted right by 22) shifted left by 28).

The store in step 1 is either zero or `0x80000000`, depending on the retained index’s least significant bit.

The word-read address ranges from 20 through 1043. It is not necessarily word-aligned. No access splitting or alignment repair is shown.

### 10. Write a zero word and two selector-mediated bytes

Four calls produce a destination index. Zero is stored as a 32-bit word at wrap(destination index × 4 + 20), followed by a dependency call receiving wrap(destination index × 4).

Four further calls produce a **late word**. The operation:

1. Loads the selector byte at wrap(terminal base + 1).
2. Stores the low byte of the late word shifted left by two at address 17 plus that selector.
3. Calls a dependency with the late word shifted left by two.
4. Obtains a 32-bit result and passes it onward together with wrap(late word shifted left by 30).
5. Makes four calls, the last producing another result.
6. Reloads the selector byte at wrap(terminal base + 1), saving this selector.
7. Stores the low byte of the late word’s field extraction at address 17 plus the saved selector.
8. Calls a dependency with the latest result shifted left by two.

The two selector-mediated stores can target different addresses.

### 11. Store through a saved selector and capture the global

Four calls occur. The operation then stores the new third argument shifted left by two as a 32-bit word at address 20 plus the saved selector.

The saved selector is not refreshed after those four calls. Dependency changes to the selector byte therefore do not change this particular destination.

Next, the unnamed 32-bit global is read and saved. A dependency is called with wrap(sixth index × 4), followed by four calls, the last producing a paired result.

The low half shifted left by two becomes the next fourth argument. The operation reloads the selector byte at wrap(terminal base + 1), then writes the low byte of the saved global logically shifted right by 22 at address 17 plus that selector.

The saved global, not a later reread, supplies this byte.

### 12. Finish the iteration and repeat

The paired result’s high half is passed to a dependency. Another result is obtained and passed onward with the new fourth argument shifted left by 28.

Four calls follow, the last producing a result. The operation reloads the selector byte at wrap(terminal base + 1), writes zero at address 17 plus it, and calls a dependency with the latest result shifted left by two.

Four final calls produce another result. Before the last dependency call:

- The register-like value for the next iteration becomes wrap(new fourth argument × 4).
- The continuation-like state is assigned a fixed internal continuation value.
- The final result shifted left by two is supplied twice to the final dependency.

Execution then begins the next iteration unconditionally.

## Side effects

The visible effects include ordered indirect calls, byte reads and writes, 32-bit reads and writes, and iteration-carried state changes.

Memory regions are not proven disjoint. A write can alias a later selector, the unnamed global, another data location, or memory used by dependencies. In particular, low-address selector-mediated writes can affect subsequent low-address accesses.

No allocation, release, ownership transfer, locking, atomic operation, or memory barrier is explicitly established. Dependency calls may perform any of these actions; their behavior is unknown.

No ownership guarantee can be inferred merely because a numeric value is used as an address. Callers must provide a valid execution context for every reached call and memory access. The full valid-input domain cannot be enumerated without the unresolved dependencies.

## Dependencies

**F-UNRESOLVED — indirect dependency class**

- Contains numerous distinct call sites with unresolved identities.
- Some receive explicit 32-bit arguments; others have no fully established interface.
- Selected results are consumed as either 32-bit quantities or paired 32-bit halves.
- No preservation of globals, memory, registers, or ownership is established.
- No error convention or guaranteed return behavior is established.
- Distinct call sites must not be merged merely because their surrounding patterns resemble one another.

**F-b930db9ce7d7 — iteration target**

The visible outer repetition re-enters the body. This is not evidence of recursive invocation or stack growth.

The continuation-derived first target and internally formed later targets require valid executable destinations. Their concrete identities and calling conventions are documented-only.

## Acceptance examples

These are conditional static acceptance cases, not executed tests.

1. **Initial word propagation.** If the fourth argument is 2 and the initial result-producing call returns 3, the first visible word store must write 12 at address 8212. If the subsequent destination index is 5, the second word store writes the saved value 12 at address 40.

2. **Byte-selected destination.** If a selector load returns 255, its associated selector-mediated byte store targets address 272. It must not target the selector’s original base plus 17.

3. **Reload across dependencies.** If the persistent-base selector is 7 at the first clear and becomes 9 during intervening calls, the corresponding clear addresses are 24 and 26. Reusing the first selector would violate the visible read sequence.

4. **Saved selector across dependencies.** If the selector saved in phase 10 is 8, the phase-11 word store targets address 28, even if intervening calls change the selector memory to 12.

5. **Shifted second-word argument.** For any 32-bit second index, the second word shifted left by 28 is zero under the stated wrapping model. A nonzero argument at that call position would contradict that model.

6. **Potentially unaligned word read.** If the selector word’s field extraction equals 1, phase 9 requests a 32-bit read at address 21. Whether that succeeds, faults, or has target-specific effects is unknown.

7. **Global snapshot.** If the saved global is `0x00400000`, the later byte derived from it is 1. A dependency changing the global after capture does not alter that saved contribution.

8. **Uninterrupted execution.** If every dependency returns and every memory access succeeds, the operation starts another iteration. Normal return after one iteration would contradict the visible control flow.

## Uncertainties

The following remain **documented_only**:

- Application-level purpose and data model.
- Concrete dependency identities and complete calling conventions.
- Valid initial continuation-like and register-like context.
- Meaning and preservation of the additional register-derived argument.
- Exact target behavior for overflow, unaligned access, inaccessible low addresses, and memory-mapped accesses.
- Endianness of 32-bit memory transfers.
- Whether any dependency reports failure, terminates execution, or changes control flow.
- Ownership, lifetime, synchronization, and concurrency requirements.
- Reliability of the recovered function boundary and interfaces. The supplied material includes analysis warnings about removed unreachable content and overlapping global symbols.

No inferred intent is required to explain the visible operations.

## Evidence

**Established static evidence:** the supplied representation contains an unconditional loop, ordered indirect calls, explicit byte and 32-bit accesses, paired-result extraction, a late global read, and updates to values carried into the next iteration.

**Interpretive model:** 32-bit wrapping arithmetic makes the width-dependent address and shift descriptions precise. It is identified explicitly rather than treated as runtime-confirmed behavior.

**Inferred intent:** repeated offset patterns may represent a common layout or protocol, but no such interpretation is established.

**Verification status:** no runtime tests have been performed. The acceptance examples describe consequences of the static representation and the stated arithmetic model, not observed executions.
