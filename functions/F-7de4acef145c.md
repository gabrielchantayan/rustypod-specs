<!-- Trusted provenance: F-7de4acef145c; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-7de4acef145c

## Purpose

F-7de4acef145c performs an ordered sequence of indirect calls, memory accesses, arithmetic-derived writes, and conditional repetition. When execution reaches its shared return tail, it produces a 64-bit result assembled from two independently derived 32-bit halves.

**Established static evidence:** The supplied representation establishes the explicit access sequence, arithmetic expressions, branch conditions, and path-dependent indirect-target calculations described below.

**Inferred intent:** The routine appears to coordinate external operations and update caller-associated and globally addressed state. No more specific application purpose is established.

This is a conditional behavioral contract, not a guarantee that arbitrary inputs execute successfully. Memory accesses must be valid, required machine state must be available, and dependencies must return normally for subsequent behavior to occur.

## Interface

### Explicit inputs

| Input | Visible representation and use |
|---|---|
| Context address | A 32-bit address-like value. The routine reads a 32-bit word at byte offset 100. |
| Opaque argument | A 32-bit value with no direct use visible in this body. |
| Output-region address | A 32-bit address used for one displaced byte store and four consecutive 32-bit stores. |
| Arithmetic input | A signed 32-bit value used in a subtraction followed by an arithmetic right shift. |

The word captured from context offset 100 is called the **context word** below. It serves as an address base, an output displacement, a signed branch operand, and an arithmetic operand. Its uses do not establish a conventional pointer or integer type.

### Implicit inputs

The visible behavior also depends on incoming state outside the four explicit arguments:

- Two 32-bit values written into the output region; one also supplies a byte store.
- A 32-bit value used to form the initial loop displacement.
- A 32-bit offset added to a common-section call argument.
- A return-address-like value used to form the first indirect target.
- Coprocessor-related state.
- Secondary register results supplied by several indirect calls.
- Fixed-location global values and a zero-tested global flag.

A callable wrapper exposing only the four explicit inputs is not demonstrably sufficient.

### Result

The returned 64-bit value has:

- **Low half:** twice a selected 32-bit value, retaining only the low 32 bits.
- **High half:** a 32-bit global word read in the shared return tail, reduced by `0x54`, retaining only the low 32 bits.

The selected low-half input is either:

1. The low half of the loop’s 64-bit-result call, on early loop exit; or
2. The arithmetic-right-shift-by-nine result of the common section’s final secondary register result, on a zero-flag exit.

No success indicator, error encoding, ownership transfer, or semantic relationship between the halves is established.

### Widths and arithmetic

Addresses and working arithmetic are represented at 32-bit width. The contract describes their bit patterns modulo \(2^{32}\) where the visible operations require truncation. Signed comparisons and arithmetic right shifts interpret those patterns as signed 32-bit values.

Byte and halfword stores retain the source’s low 8 and 16 bits respectively. A 64-bit call result is separated into low and high 32-bit halves.

This machine-width interpretation is not a claim that analogous signed arithmetic would be defined in every source language. Address-space wrapping, alignment requirements, endianness, and access-fault behavior remain **documented_only**.

## Behavior

### Initial acquisition and output writes

The routine first reads the context word from context offset 100.

It then performs a coprocessor transfer using that word, fixed selector values 3, 3, and 2, and incoming coprocessor-related state. The operation’s architectural meaning is unresolved.

Next, it reads a 32-bit word at the context-word address plus `0x14`. It calls an indirect target formed by adding `0xCA6` to the incoming return-address-like value. No low-bit clearing is visible for this target. The visible arguments are the original context address and the newly read word.

Call the 32-bit return the **initial result**. After this call returns, the routine performs the following stores in order:

1. One byte at output address plus context word, using the low byte of an implicit incoming value.
2. The initial result at output offset 0.
3. The output address, represented as 32 bits, at output offset 4.
4. Another implicit incoming value at output offset 8.
5. The full 32-bit value that supplied the first byte store, at output offset 12.

The displaced byte store need not lie within the first 16 output bytes. If it overlaps a later word store, that later store determines the overlapping bytes, subject to the target memory model.

### Initial arithmetic call and branch selection

The routine arithmetically shifts the initial result right by nine and records whether the unsigned interpretation of that shifted value is greater than `0xFFFFFFBE`.

Under the stated 32-bit interpretation, this flag is set exactly when the signed initial result is between **−33,280 and −1 inclusive**.

An indirect call then receives:

1. The initial result.
2. Its arithmetic-right-shift-by-nine value plus `0x41`.
3. The arithmetic input reduced by `0x3C` at 32-bit width, then arithmetically shifted right by 20.

This call supplies a 64-bit result.

If the recorded flag is clear, execution enters the common section using that result.

If the flag is set, the routine first stores the context word’s low 16 bits at the returned low-half address plus `0x2A`. It then compares these Boolean facts:

- The context word is negative as a signed 32-bit value.
- Signed subtraction of `0x2F` from the returned high half overflows.

Equal Boolean values select the common section. Unequal values select direct loop-body entry.

This is not a signed comparison of the returned high half against `0x2F`: one operand of the Boolean comparison comes from the context word. Subtracting `0x2F` overflows only for signed high-half values from −2,147,483,648 through −2,147,483,602 inclusive.

For direct loop-body entry:

- The working displacement becomes the implicit subtraction input minus the context word.
- The working selector becomes the returned low half logically shifted right by 25.
- The working state becomes the returned high half.
- The loop-entry target base remains the initial value `0x08573355`.

### Loop-entry target and initial loop work

The loop-entry indirect call has a **path-dependent target**, not merely path-dependent arguments.

| Entry path | Target base |
|---|---:|
| Direct entry from initial branch selection | `0x08573355` |
| Entry after any nonzero common-section flag | `0x08573459` |

In both cases, target formation first clears the base’s low two bits and then adds `0x4A8`. Consequently:

- Direct initial loop entry calls target `0x085737FC`.
- Entry after a nonzero common-section flag calls target `0x08573900`.

The second base is installed before the common section’s third call. It is therefore the base used when that section subsequently repeats into the loop, whether the common section was reached from initial branch selection or after a completed loop body.

The loop-entry call receives:

1. The working selector.
2. The working state.
3. An address into the current stack frame.

A second indirect call follows and supplies a 64-bit result. The routine then:

1. Computes the working state plus ten at 32-bit width.
2. Stores the current working displacement as a 32-bit word at working-state address plus `0x3E`.
3. Tests whether the low-32-bit product of the incremented state and `0x10000` is negative.

The final test is equivalent to testing bit 15 of the incremented state. It does not test that state’s own sign bit.

If the test is true, execution immediately enters the shared return tail. The selected low-half input is the low half of the immediately preceding 64-bit call result. The displacement store has already occurred; the remaining loop-body stores and common-section calls do not occur.

### Remaining loop-body work

If the early-return test is false, execution continues in this order:

1. Store the preceding 64-bit result’s low 16 bits at its high-half address plus `0x3C`.
2. Invoke an indirect operation with no explicit arguments visible.
3. Invoke another indirect operation with no explicit arguments visible, obtaining a 32-bit address-like result.
4. Store the halfword `0x0026` at that address plus 2.

The routine then forms a **shifted quantity** by multiplying the incremented state by `0x10000000` at 32-bit width and arithmetically shifting the resulting signed bit pattern right by nine. It adds this quantity to the incremented state, retaining 32 bits.

Next, it captures a fixed-location 32-bit global mixing word and invokes another indirect operation with:

- Twice the address-like result from step 3, at 32-bit width.
- The incremented state multiplied by `0x10000`, then reduced by `0x51`, at 32-bit width.

After the call, the routine XORs the previously captured mixing word with the previously formed sum. The result is the **derived address**. The operand is the captured global value, not a fresh post-call read.

A **derived value** is formed by adding `0x2F` to the shifted quantity. The routine then:

1. Stores the derived address in a stack slot at byte offset `0x130` from the represented stack base.
2. Reads an unsigned byte at derived address plus 1.
3. Calls an indirect operation with a fixed address-valued token, that byte, the derived value, and a secondary result from the preceding two-argument call.
4. Stores the derived value as a 32-bit word at derived address plus `0x24`.
5. Invokes another indirect operation with no explicit arguments visible.

The fixed token passed in step 3 has value `0x085734E8`; no local dereference of that token is visible.

For the following common section, the low half is the derived address. The high half is the final call’s secondary 32-bit result plus `0x30`.

### Common processing and repetition

The common section receives either the initial arithmetic call’s 64-bit result or the composite produced by a completed loop body.

From its current halves, it computes:

- The next selector: twice the low half, retaining 32 bits.
- The next displacement: the square of twice the high half, retaining 32 bits.

Three indirect calls follow:

1. A call receiving the current low half plus an implicit incoming offset.
2. A call receiving a freshly read fixed-location 32-bit global word.
3. A call with no explicit arguments visible, supplying the secondary result subsequently used for exit or repetition.

The first call’s target is path-dependent. Its base is `0x08573355` on initial common-section entry and `0x08573405` after a completed loop body; `0xAAA` is added without visible low-bit clearing.

The second call’s target clears the low two bits of base `0x08573449` before adding `0xAB4`.

Before the third call, the loop-entry target base is set to `0x08573459`. This update matters to a subsequent repetition: the loop-entry call does not reuse the initial base.

After the three calls, the routine reads a fixed-location global flag:

- **Zero:** Arithmetically shift the third call’s secondary 32-bit result right by nine and enter the shared return tail.
- **Nonzero:** Store the next displacement at twice the current high half plus `0x34`; make the third call’s secondary result the working state; retain the computed next selector and displacement; enter the loop using the updated loop-entry target base.

There is no visible iteration bound. Continued nonzero flag reads and false early-return tests can permit unbounded repetition.

### Shared return tail

The shared tail reads its fixed-location 32-bit global word after the path-specific operations already described. It subtracts `0x54` for the high result half, doubles the selected low-half input, and returns their 64-bit combination.

An earlier global value is not substituted for this tail-time read.

## Side effects

Visible effects include:

- The initial coprocessor operation.
- The displaced output byte store and four output word stores.
- Conditional halfword and word stores through call-result-derived and arithmetic-derived addresses.
- Exposure of a stack address to an indirect dependency.
- A later stack-slot write at offset `0x130`.
- Reads of separate fixed-location global words and a flag.
- Indirect calls whose additional effects are unknown.

No allocation, deallocation, reference counting, or rollback is directly visible. This does not establish that dependencies perform none of these actions.

No ownership transfer or pointer-retention permission is established. In particular, passing stack storage does not prove that the dependency uses it only during the call.

Aliasing is not excluded. Output storage, context storage, globals, stack-accessible storage, and derived addresses may overlap. Earlier stores or dependency effects can therefore influence later reads where addresses alias.

The routine does not visibly undo completed stores if a later call or memory access fails. No exception-handling or error-recovery contract is established.

## Dependencies

**F-882dff43a182** identifies the shared return-tail location in the supplied mapping. The visible transfer reaches that location directly; an independently invoked function boundary is not established.

No opaque function IDs are supplied for the other indirect targets. Their identities remain **documented_only** rather than being assigned fabricated IDs. Their observable call roles include:

- Initial context-associated call.
- Initial three-argument arithmetic call.
- Path-dependent loop-entry call receiving stack storage.
- Loop 64-bit-result call.
- Loop operations supplying an address or secondary register results.
- Derived-address byte-consuming call.
- Three common-section operations.

The absence of dependency IDs does **not** make established target calculations unknown. In particular, the loop-entry target changes between direct initial entry and repetition exactly as specified above.

Visible argument lists and secondary register results do not establish complete declared signatures. Dependencies may consume additional machine state, mutate globals, retain addresses, allocate or release memory, fail, or never return. None of those behaviors may be presumed absent.

## Acceptance examples

These are conditional deductions from the static representation, not executed tests. Each assumes valid accesses and normally returning dependencies up to the event described.

1. **Initial flag boundaries.** An initial result of −33,280 sets the flag; −33,281 clears it. An initial result of −1 sets it; zero clears it.

2. **Output overlap.** With a zero context word, the displaced byte store targets output offset 0. The subsequent initial-result word store overwrites that byte according to target byte order.

3. **Store before branch.** Whenever the initial flag is set, the halfword store at the arithmetic call’s low-half address plus `0x2A` occurs before deciding between common processing and direct loop entry.

4. **Mixed-origin branch and initial target.** If the returned high half is zero, subtracting `0x2F` does not overflow. A negative context word therefore selects direct loop entry. That entry clears the low two bits of `0x08573355` and adds `0x4A8`, selecting target `0x085737FC`.

5. **Repeated-entry target.** If the common-section flag is nonzero, the ensuing loop-entry call uses base `0x08573459`, regardless of how the common section was reached. Clearing its low two bits and adding `0x4A8` selects `0x08573900`, not the direct-entry target.

6. **Early-return ordering.** If the incremented working state is `0x00008000`, the early-return test is true. The word store at the original working-state address plus `0x3E` nevertheless occurs first. An incremented state of `0x00007FFF` does not trigger this exit.

7. **Captured global operand.** If the two-argument loop dependency changes the global mixing word, the subsequent XOR still uses the value captured before that call.

8. **Zero-flag result.** If the common section’s final secondary result is −513 and the subsequently read flag is zero, the arithmetic shift yields −2. The returned low half is `0xFFFFFFFC`.

9. **Tail-time global read.** If the return-tail global word is `0x00000020` when read, the high return half is `0xFFFFFFCC`, independently of which exit selected the low-half input.

10. **Repetition without another flag read.** A nonzero flag causes the common-section word store and another loop entry. If that iteration takes the early-return path, it returns without another common-section flag read.

## Uncertainties

The following remain **documented_only**:

- Valid address ranges, object extents, alignment, endianness, and fault behavior.
- Complete calling convention and sources of implicit incoming values.
- Architectural interpretation of tagged or low-bit-bearing call addresses.
- Identities and full behavior of indirect dependencies.
- Coprocessor semantics, privilege requirements, and failure behavior.
- Ownership, lifetime, retention, and release responsibilities for all addresses.
- Whether stack storage escapes or is accessed asynchronously.
- Dependency effects on globals, other memory, and implicit machine state.
- The global flag’s exact access width where not explicitly typed.
- Concurrency, synchronization, and atomicity requirements.
- Exact correspondence between recovered signed expressions and executable instruction semantics.
- Completeness of the recovered control-flow and shared-tail boundaries.

These uncertainties do not erase established sequencing or target variation. They limit claims about execution validity, external effects, and higher-level intent.

## Evidence

The evidence is solely the supplied static representation and opaque dependency mapping. It supports the explicitly represented memory widths, operation ordering, conditional stores, arithmetic expressions, target-base assignments, low-bit clearing, repetition, and return construction.

The representation exposes incomplete incoming state and overlapping-global warnings. These constrain confidence in source-level types, object boundaries, and conventional callable interfaces.

No runtime tests, execution traces, implementation inspection beyond the supplied representation, or dynamic validation have been performed. Acceptance examples are conditional static deductions, not observed executions.
