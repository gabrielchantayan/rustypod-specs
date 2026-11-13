<!-- Trusted provenance: F-dd4470ab4dfe; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-dd4470ab4dfe

## Purpose

F-dd4470ab4dfe performs a conditionally selected 16-bit memory transfer, evaluates a sequence of address and machine-state conditions, and enters an unbounded sequence of calls and software interrupts.

No application-level purpose is established. The supplied representation does not support describing this function as a conventional buffer operation, dispatcher, or initialization routine. Its behavior depends on entry machine state beyond the five declared arguments.

Under the 32-bit arithmetic interpretation described below, much of the later conditional memory-manipulation region is unreachable. This contract distinguishes those reachability conclusions from operations merely present in the supplied representation.

There is no represented normal return. Execution can nevertheless stop progressing if a memory access faults, a called target does not return, or the software interrupt transfers control permanently.

## Interface

### Declared arguments

The represented interface has five arguments:

| Position | Represented width | Established use |
|---|---:|---|
| First | 32 bits | No explicit value use |
| Second | Unsigned 32 bits | Controls initial selection and derived address values |
| Third | 32 bits | No explicit value use |
| Fourth | 32 bits | No explicit value use |
| Fifth | 16 bits | Its address participates in arithmetic; its supplied value is not explicitly read |

“Unused value” does not mean the argument can be removed from a compatible interface. In particular, changing argument placement could change the address associated with the fifth argument.

The declared result is void. No result value or successful-return convention is established.

### Additional entry state

The representation also consumes entry values that are not among the declared arguments:

- A 32-bit word used in equality conditions.
- A second 32-bit word used as an unsigned byte displacement and in equality conditions.
- A pointer used as the source of a 16-bit read.
- A 32-bit word used as a subtractive byte displacement.
- Another 32-bit word used in an addition.
- A pointer used as an initial execution-base value.
- In a later represented region, an additional machine-state value whose ordinary interface meaning is not established.

For clarity, this contract calls these values the **comparison word**, **forward displacement**, **transfer source**, **backward displacement**, **auxiliary word**, and **entry base**, respectively. These names describe roles, not additional declared parameters.

Let the **argument address** mean the address of the fifth argument’s 16-bit storage.

### Width and layout model

The supplied representation uses 16-bit halfwords and 32-bit words and addresses. Reachability conclusions below use:

- Arithmetic modulo \(2^{32}\) for address-sized additions and multiplications.
- Unsigned comparisons where unsigned values are represented.
- Two-byte scaling when advancing a pointer to a 16-bit element.
- Byte scaling where an address is explicitly treated as an integer.
- Logical right shifts of unsigned words.

No byte order, stack layout, pointer tagging rule, instruction-set mode, or alignment-fault policy is established.

Addresses derived from integers are not thereby valid memory or executable addresses. Validity is an external precondition for whichever path actually executes.

## Behavior

### Initial selection

The second argument is multiplied by \(2^{28}\), retaining the low 32 bits.

Consequently:

- The product is zero exactly when the second argument’s low four bits are zero.
- Higher bits of the second argument do not change that selection.
- Bit 2 of the second argument is also examined, but it is necessarily clear on the path where the low four bits are zero.

If the low four bits are nonzero, the initial 16-bit transfer is skipped. The subsequent conditional sequence does not enable any of the intervening memory-write regions. The first recurring-call base remains the entry base.

If the low four bits are zero, the function performs the transfer described next.

### Conditional 16-bit transfer

On the zero-low-nibble path:

1. The destination address is formed by adding the forward displacement to the transfer-source address, with byte—not halfword—scaling.
2. One unsigned 16-bit value is read from the transfer source.
3. The working transfer-source address is changed to destination minus backward displacement.
4. The saved 16-bit value is written to the destination.
5. The entry base becomes the value used to begin the next conditional sequence.

The read precedes the write. Therefore, source and destination may coincide without changing which value is written. Partial overlap and unaligned addresses remain subject to the underlying memory model.

This is one halfword transfer, not a loop or a transfer whose length is specified by an argument.

The update of the working transfer-source address is internal state in the supplied representation. There is no explicit store of that updated pointer back to caller-owned pointer storage.

### Conditional progression after the transfer

The next conditional region can advance only when the entry base is zero.

If the entry base is nonzero, the transfer, if selected, is followed by the recurring-call sequence using that entry base.

If the entry base is zero, subsequent progression requires several zero-result and equality conditions. The repeated shift expressions in this region are reached with a zero shift count: their count comes from a value already required to be zero. They therefore do not require choosing a policy for large shift counts on these reachable paths.

Under the stated 32-bit model, reaching the exceptional memory update below requires all of the following:

- The second argument’s low four bits are zero.
- The entry base is zero.
- The argument address shifted right by one is zero.
- The comparison word equals the forward displacement.
- The updated transfer-source address equals the backward displacement.
- The auxiliary word is zero.
- The argument address plus twice the forward displacement is zero modulo \(2^{32}\).

These are conditions on actual entry state and storage addresses, not validations performed through a public error interface.

In particular, the shifted-address condition restricts the argument address to numeric address 0 or 1. A normal aligned, nonzero stack location would not satisfy it. The representation does not establish an ABI or memory map that makes this exceptional path feasible.

### Exceptional halfword update and base change

When the entire preceding condition sequence succeeds, the representation:

- Derives a new entry base by adding 64 times the argument address to the old entry base, modulo \(2^{32}\).
- Contains a write of 64 times the argument address to an unresolved machine-state destination.
- Reads a 16-bit value through the old entry base and uses its zero-extended value as the address of a 16-bit write.
- Writes a value obtained by adding 15 to a 16-bit read through that same old entry base, retaining the low 16 bits.
- Advances an internal 16-bit-element pointer by one element, a two-byte change.

The halfword write expression contains two reads through the old entry base: one determines the destination address, and one determines the value to increment. Their relative evaluation order is not established by the supplied representation. A replacement must not silently promise that this is a single read, especially for volatile or externally changing storage.

The old entry base must be zero to reach this region. Thus, this path requires a readable halfword at numeric address zero before it can complete. If the address-derived destination overlaps that location, both reads still supply inputs to the write; no later read-back is represented.

The unresolved machine-state write is not sufficiently characterized to describe it as a normal global assignment, a stack relocation, or an ownership operation.

### Reachability of the remaining conditional region

The supplied representation contains additional conditional address arithmetic, reads, and writes after the exceptional update. Under the 32-bit model, that region cannot reach its substantive operations:

- Without the exceptional update, the fresh condition enabling the next chain is false.
- With the exceptional update, reaching the next chain requires the new entry base to be zero.
- The preceding conditions already force the first derived address and the half-shifted argument address to zero.
- The exceptional update has advanced the relevant halfword pointer from zero to numeric address 2.
- The next equality-producing operation therefore produces 2 rather than zero, preventing further progression.

Accordingly, the later represented pointer-sized stores, signed-halfword-derived stores, and additional address adjustments are not required observable effects of any path under this interpretation.

This conclusion is conditional on the supplied control flow and word-width model being faithful. It is not evidence that the original executable has been tested or that the representation accurately captures its intended entry boundary.

### Recurring calls and software interrupt

After the conditional regions, execution enters a loop with this observable order:

1. Call the execution address obtained by adding `0xEB6` bytes to the current base.
2. Supply three represented 32-bit arguments: 0, 0, and `0xFFFFFFA5`.
3. If that call returns normally, call execution address `0x0857A94B`, with no explicit arguments represented.
4. If that call returns normally, issue software interrupt `0x5B`.
5. If the interrupt resumes the represented continuation, set the base to `0x083D4CC2` and repeat.

The first call is to the computed execution address itself. The representation does not show reading a function pointer from memory at that address.

The third argument’s bit pattern is `0xFFFFFFA5`; its signed 32-bit interpretation is −91. No dependency signature establishes which interpretation the target uses.

The first iteration uses the entry base, except where the exceptional path changes it. Every later iteration that reaches the base reset calls the first target at `0x083D5B78`.

No loop exit, return, timeout, or success indication is represented.

## Side effects

### Established explicit effects

Depending on path and successful completion of earlier operations:

- A 16-bit read followed by a 16-bit write for the initial transfer.
- On the exceptional path, two represented halfword reads through address zero and a halfword write through an address derived from one of those reads.
- An unresolved machine-state write on that exceptional path.
- Repeated transfers of control to two executable targets.
- Repeated software interrupts.

Call targets and the interrupt may produce additional effects. Their effects are not bounded by the visible operations in this function.

### Ownership and preservation

No explicit allocation, release, ownership transfer, reference-count operation, or resource cleanup is shown.

That absence does not establish that dependencies leave memory allocated, preserve globals, or refrain from releasing resources. Likewise, this representation does not establish preservation of entry machine state beyond what a real calling convention might require.

### Errors and prerequisites

There is no explicit status result, exception handler, rollback, or recovery branch.

Completion of a selected path requires:

- Readable memory for every selected read.
- Writable memory for every selected write.
- Executable targets at the computed call addresses.
- Compatible calling conventions.
- A defined software-interrupt mechanism.

The contract does not invent behavior for unmapped, misaligned, non-executable, or otherwise invalid addresses. Failure may prevent all subsequent operations.

## Dependencies

No dependency function IDs are supplied for the two call targets. Assigning identities to them would invent provenance. They are therefore documented as unresolved execution dependencies:

| Dependency | Inputs visible here | Established continuation condition |
|---|---|---|
| Base-relative execution target | Three 32-bit values: 0, 0, `0xFFFFFFA5` | Must return for the fixed-address call to occur |
| Fixed execution target at `0x0857A94B` | No explicit arguments represented | Must return for the interrupt to occur |
| Software interrupt `0x5B` | Interrupt selector; other machine-state inputs unspecified | Must resume the represented continuation for repetition |

The absence of explicit arguments to the fixed target does not establish that it consumes no registers, memory, or other ambient state.

## Acceptance examples

These are conditional static acceptance criteria, not executed tests.

1. **Transfer skipped by the low nibble.**  
   For a second argument of 1 and an entry base of `0x1000`, no explicit initial halfword read or write occurs. The first call target is `0x1EB6`. Subsequent behavior depends on that target returning.

2. **High bits do not prevent the transfer.**  
   For a second argument of `0x80000000`, its low nibble is zero. If the transfer source is `0x2000` and the forward displacement is 6, the function reads a halfword at `0x2000` and writes the saved value at `0x2006`.

3. **Transfer overlap preserves read-before-write ordering.**  
   With a zero forward displacement, the selected transfer reads the source halfword before writing that value back to the same address. No second read is needed for this initial transfer.

4. **Nonzero entry base stops exceptional progression.**  
   After a selected transfer, a nonzero entry base prevents the exceptional address-zero update. The recurring-call sequence begins using that base.

5. **Ordinary argument storage blocks the exceptional update.**  
   With a zero entry base but an argument address of `0x1000`, shifting the argument address right by one yields `0x800`, not zero. The exceptional update is not reached.

6. **Iteration advances only after the interrupt continuation.**  
   If both targets return and interrupt `0x5B` resumes execution, the next iteration’s first target is `0x083D5B78`, regardless of the first iteration’s base.

7. **Nonreturning dependency prevents later effects.**  
   If the base-relative target never returns, neither the fixed-address call nor the software interrupt is reached on that iteration.

## Uncertainties

The following are **documented_only**:

- The true calling convention and complete entry-state interface.
- Whether the represented entry boundary corresponds to an intended callable routine.
- The meaning and validity of the unresolved machine-state destination.
- Byte order, alignment behavior, instruction-set mode, and pointer representation.
- The identities, signatures, ownership effects, and global-state effects of execution dependencies.
- The interrupt’s inputs, outputs, failure modes, and resumption behavior.
- Relative evaluation order of the two reads in the exceptional halfword-write expression.
- Any behavior requiring a different arithmetic model from the stated 32-bit interpretation.
- Whether any environment makes the exceptional address-zero path usable.

No functional intent beyond the established effects is inferred.

## Evidence

The evidence is limited to the supplied static representation.

Established static evidence includes the five declared argument widths, the selected halfword transfer, the zero-gated condition sequences, the represented exceptional update, the two call expressions, the interrupt selector, and the unconditional repetition structure.

The low-nibble selection, two-byte pointer scaling, conditional unreachability result, and repeated target address are deductions from those expressions under the explicitly stated 32-bit model.

No runtime tests, executable inspection, dependency inspection, or external ABI verification have been performed.
