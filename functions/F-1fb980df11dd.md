<!-- Trusted provenance: F-1fb980df11dd; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-1fb980df11dd

## Purpose

F-1fb980df11dd is a non-returning sequence of indirect dependency invocations, memory reads, and memory writes, repeated without an explicit exit condition.

The established behavior is the ordering of those operations and their data relationships. A higher-level purpose—such as initialization, device servicing, or dispatch—is not established. In particular, several values are used both as numeric data and as address components; this contract does not reinterpret them as validated handles or ordinary object pointers.

Each iteration includes a software breakpoint with immediate value `0x92`. Execution beyond that point is conditional on the execution environment permitting continuation.

## Interface

**Explicit inputs:** None.

**Explicit outputs:** None. There is no ordinary return path in the supplied representation.

**Implicit inputs:**

- An incoming link-related value used to derive the first indirect call target.
- An incoming preserved-register value used in a late byte-store address.
- A frame-like base used for several word accesses and explicitly reduced by `0xF4` during each completed iteration.
- Two global cells containing 32-bit address-like values.
- Memory reachable through loaded values, dependency results, and calculated addresses.
- Dependency results appearing in primary and secondary result channels.

The meaning and calling convention of the implicit register state are not established. These are necessary execution inputs, not a proposed public API.

**Widths and notation:**

- Byte: 8 bits.
- Halfword: 16 bits.
- Word: 32 bits.
- Paired result: represented as 64 bits, split into low and high 32-bit halves.
- All offsets below are byte offsets.
- A halfword read used as a numeric address component is zero-extended to 32 bits.
- A store narrower than its source retains the low bits of that source.

Names such as “first context base” and “later pair” identify values within this contract; they do not assert source-level types or object identities.

The memory byte order, alignment requirements, and precise treatment of overflowing signed expressions are not established. For the stated shift and address relationships, the relevant values are 32-bit quantities. Any acceptance example involving arithmetic presupposes values and an execution model for which the stated operations are defined.

## Behavior

### 1. Start the iteration and capture the first context

At the start of each iteration, the value later written as a cleared byte is set to zero.

The function invokes dependencies in this order:

1. **F-D01**, with two explicit arguments, both zero.
2. **F-D02**, with no explicit arguments shown.
3. **F-D03**, with no explicit arguments shown.

It then reads a word from the first global address cell and retains that value as the **first context base**.

Next:

4. Invoke **F-D04**.
5. Invoke **F-D05**, obtaining a paired result.

Let the low and high words of this pair be the **first low word** and **first high word**. Invoke **F-D06** with:

- First argument: the first low word.
- Second argument: the sum of the first high word and first low word.

A zero-valued byte also appears as an addend in the represented computation; it does not change the numeric sum.

After F-D06, store one byte from a secondary result channel at:

- First context base + `0x11`.

Invoke **F-D07**, then store a zero byte at:

- First context base + `0x15`.

The first context base remains the value captured before F-D04. A dependency’s modification of the global cell does not, by itself, replace that already captured value.

### 2. Store early results and invoke the middle dependency chain

Invoke **F-D08**. Store its halfword result at:

- A secondary-result-derived base + `0x0C`.

The representation associates this base with a secondary result channel at this call boundary. Its validity and relationship to the halfword result are unknown.

Invoke **F-D09**, obtaining another paired result. Then, in order:

1. Store a zero halfword at its high word + `0x2C`.
2. Store its low word, logically shifted right by 26 bits, as a word at frame base + `0x26C`.

The latter stored value lies between zero and 63.

Establish the constant `0x21C48960` for later operations, then:

3. Invoke **F-D10**.
4. Read a word at frame base + `0x320`; retain it as the **saved destination base**.
5. Invoke **F-D11**.
6. Invoke **F-D12**.
7. Store the low word returned by F-D09, shifted left by two bits, at saved destination base + `0x54`.

The saved destination base is captured before F-D11 and F-D12. The store uses that captured address component, not a later reread of the frame slot.

Invoke, in order:

8. **F-D13**.
9. **F-D14**.
10. **F-D15**.
11. **F-D16**.
12. **F-D17**.

Now read a halfword at absolute address `0x21C48994`, which is `0x21C48960 + 0x34`. Retain its zero-extended value as the **first halfword-derived base**.

Invoke **F-D18**.

Read a word at frame base + `0x280` and pass that word as the explicit argument to **F-D19**. Retain F-D19’s 32-bit result.

Execute the software breakpoint with immediate `0x92`.

There is no supplied handler, recovery branch, or alternate result for this breakpoint. All remaining behavior assumes execution resumes after it.

### 3. Perform indexed halfword stores

Scale F-D19’s result by four for use as an address offset.

Separately derive a **fixed derived base** from the earlier constant:

- Shift `0x21C48960` right by 18 bits.
- Multiply the result by four.
- The resulting base is `0x21C4`.

Read a word at first halfword-derived base + `0x44`. Retain it as the **main destination base**.

Then:

1. Invoke **F-D20**.
2. Invoke **F-D21**, obtaining a word.
3. Read a word at first halfword-derived base + `0x04`.
4. Invoke **F-D22** with the F-D21 result as its first explicit argument and the just-read word as its second.

Read the second global address cell and retain its word as the **table base**.

Invoke **F-D23**.

Store a halfword at:

- Table base + four times the result of F-D19.

The stored halfword is the low 16 bits of the table base itself. It is not the F-D19 result.

Continue:

5. Invoke **F-D24**.
6. Read a word at fixed derived base + `0x54`, equivalent to address `0x2218`. Retain it as the **intermediate base**.
7. Invoke **F-D25**, obtaining an address-like result.
8. Store a halfword at that result + `0x04`.

This second halfword store contains the low 16 bits of the fixed derived base: `0x21C4`.

Invoke **F-D26**, followed by **F-D27**.

Read a halfword at intermediate base + `0x34`. Zero-extend it and retain it as the **selector**.

### 4. Consume the later pair and change the frame-like base

Invoke **F-D28**, obtaining a paired result. Let its words be the **later low word** and **later high word**.

Perform these operations in order:

1. Store the later low word as a word at frame base + `0x258`.
2. Read a halfword at later high word + `0x14`; zero-extend it and retain it as the **halfword payload**.
3. Form a scaled value from the later low word shifted left by six bits.
4. Form another scaled value from the later high word multiplied by four.
5. Reduce the frame-like base by `0xF4`.
6. Form a selector offset equal to the selector multiplied by four.
7. Store the later low word at selector offset + `0x74`.

The frame-slot store occurs before the explicit frame-base reduction. The selector-indexed store is not expressed relative to that frame base: its represented address is `4 × selector + 0x74`.

No matching frame-base restoration is visible before the next iteration. If the supplied representation is executed literally, the reduction accumulates across completed iterations. Whether this represents an actual stack operation or an imperfectly reconstructed register role is unknown.

### 5. Complete the late stores and repeat

Invoke **F-D29**, then store the halfword payload as a 32-bit word at:

- Main destination base + `0x54`.

Invoke **F-D30**, then store the same zero-extended payload as a 32-bit word at:

- Main destination base + `0x64`.

These are separate stores with a dependency invocation between them. They are not an atomic paired update.

Invoke **F-D31**.

Store a halfword at:

- The later low word shifted left by six bits, plus a secondary-result-derived base associated with this part of the sequence.

The stored value is the low 16 bits of the main destination base.

Invoke **F-D32**, followed by **F-D33**.

Read one byte at:

- Selector + selector offset.

Because the selector offset is four times the selector, this is numerically `5 × selector`, subject to the applicable address arithmetic. It is not a read from the earlier table base.

Invoke **F-D34**.

Store that captured byte at:

- Halfword payload + incoming preserved-register value + `0x19`.

The byte is captured before F-D34 and stored afterward. A change to the source memory during F-D34 does not replace the captured byte.

Invoke **F-D35**.

Store the later high word multiplied by four as a word at:

- Main destination base + `0x44`.

Invoke **F-D36**.

Finally:

1. Read a word at the scaled later-high-word address + `0x44`.
2. Invoke **F-D37** with that word as its explicit argument.
3. Begin another iteration.

The first call target in the next iteration is derived from the link-related value explicitly established near the end of the previous iteration. It is therefore not justified to describe every iteration’s first target as depending solely on the original incoming link state.

## Side effects

The established direct effects include:

- Repeated indirect calls.
- Reads of two global address cells.
- Reads and writes at absolute, frame-relative, loaded, and calculated addresses.
- Byte, halfword, and word stores with the ordering described above.
- A software breakpoint on every iteration that reaches it.
- An explicit reduction of the frame-like base on every iteration that reaches the late phase.
- No explicit normal return.

The function does not show bounds checks, null checks, address validation, synchronization, rollback, or status-based error handling.

Memory regions are not proven disjoint. If two calculated addresses overlap, the actual read/write order determines which values are observed. Dependencies may also alter memory between direct accesses. Captured values remain the operands shown by the representation, while subsequent explicit reads obtain memory contents at their own execution points.

No direct allocation or release operation is identified. That does not establish absence of allocation, release, ownership transfer, or lifetime changes inside dependencies.

## Dependencies

**F-D01 through F-D37** are local opaque dependency labels assigned in invocation order within one iteration. They are not externally resolved function identities. Different labels may identify the same executable target.

Their established interface categories are:

| Labels | Established use |
|---|---|
| F-D01 | Two explicit zero arguments |
| F-D05, F-D09, F-D28 | Paired results consumed as two 32-bit words |
| F-D06 | Two computed word arguments; secondary byte result consumed |
| F-D08 | Halfword result and secondary address-like value consumed |
| F-D19 | One explicit word argument; result used as a scaled offset |
| F-D21 | Word result passed onward |
| F-D22 | Two explicit word arguments |
| F-D25 | Result used as a halfword-store base |
| F-D31 | Secondary address-like result consumed |
| F-D37 | One explicit word argument |
| Remaining labels | Invocation established; no explicit arguments or primary results used in the supplied representation |

“No explicit arguments shown” does not establish an argument-free machine-level interface. Implicit register arguments, additional result channels, clobbers, exceptional exits, and non-returning behavior remain unknown.

No dependency is assumed to preserve globals, retain object lifetimes, own memory, or release memory.

## Acceptance examples

These are conditional acceptance criteria derived from static evidence, not executed tests.

1. **First paired-result argument construction.**  
   If F-D05 supplies low word `3` and high word `4`, F-D06 receives explicit arguments `3` and `7`. The byte store at first context base + `0x11` follows F-D06; the zero-byte store at +`0x15` follows F-D07.

2. **Six-bit extraction and scaled store.**  
   If F-D09’s low word is `0x84000003`, the word written at frame base + `0x26C` is `33`. Under 32-bit left-shift truncation, the later word stored at saved destination base + `0x54` is `0x1000000C`. The two writes remain separated by F-D10, the saved-base read, F-D11, and F-D12.

3. **Breakpoint boundary.**  
   If the environment stops execution at breakpoint `0x92`, none of the operations described after that breakpoint occur during that execution. If it resumes normally, the next described phase is eligible to execute.

4. **Halfword content is not the index.**  
   If the table base is `0x12345000` and F-D19 returns `2`, the halfword store is at `0x12345008` and contains `0x5000`, assuming valid, non-overflowing address calculation.

5. **Zero-extension of the payload.**  
   If the halfword read at later high word + `0x14` is `0xFFFF`, the word stores at main destination base + `0x54` and +`0x64` contain `0x0000FFFF`, not `0xFFFFFFFF`.

6. **Captured-byte behavior.**  
   If the selector is `3`, the late byte read addresses `15`. If it reads `0xA7`, the later byte store writes `0xA7` even if F-D34 changes address `15` before returning, provided execution reaches the store normally.

7. **No normal termination.**  
   If all invoked dependencies return, all accesses succeed, and breakpoint handling resumes execution, the function begins another iteration rather than returning a success value.

## Uncertainties

The following are **documented_only**:

- Semantic purpose and domain interpretation.
- Public calling convention and required entry register state.
- Concrete identities and contracts of indirect dependencies.
- Reliability of secondary-result reconstruction.
- Whether the frame-like register is a true stack pointer.
- Byte order, alignment rules, memory-map validity, and access permissions.
- Exact overflow semantics for signed-looking shifts and address expressions.
- Whether accesses target ordinary memory or memory-mapped devices.
- Breakpoint handling and any continuation mechanism.
- Dependency-induced faults, ownership changes, exceptions, or nonlocal exits.
- Concurrency and whether external agents can change accessed memory.

Successful completion of an iteration requires readable and writable addresses of the indicated widths, callable targets, returning dependencies, and continuation after the breakpoint. Those are execution preconditions, not checks performed by this function.

## Evidence

**Established static evidence:** The supplied representation contains an unconditional loop, 37 indirect invocation sites per iteration, explicit-width memory accesses, a software breakpoint, and the data dependencies and access ordering described above.

**Inferred intent:** Repeated servicing or state propagation is plausible from the loop and distributed writes, but no such interpretation is required by this contract.

**Unknown:** Whether the supplied representation faithfully recovers all calling conventions, register roles, arithmetic semantics, and executable control flow.

No runtime tests, breakpoint experiments, memory traces, or dependency executions have been performed.
