<!-- Trusted provenance: F-3df9e5a6cc72; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-3df9e5a6cc72

## Purpose

F-3df9e5a6cc72 specifies a repeating sequence of indirect calls, scalar memory accesses, implicit execution-state updates, and a software breakpoint.

**Established static behavior:** the supplied representation contains an unconditional loop, with no normal return or conditional exit. An iteration coordinates accesses through fixed-address pointer slots, captured addresses, paired call results, and a frame-like execution base.

**Inferred intent:** the accesses may update related records or tables. No application-level purpose, record schema, or subsystem identity is established.

**Execution qualification:** progress requires calls to return, memory accesses to succeed, and execution to continue past the breakpoint. Static evidence does not establish that a complete iteration is reachable in a real execution environment.

## Interface

### Explicit interface

- No explicit parameters are shown.
- No return value is shown.
- No normal completion path is shown.

### Implicit inputs and storage bindings

The function consumes execution state and memory not supplied through explicit parameters:

| Input or storage | Established role |
|---|---|
| Incoming link-like value | Determines the first iteration’s first indirect-call target by adding 0xffe. |
| Frame-like execution base | Supplies the base for several word accesses; later decreases by 0xf4. |
| Incoming register-held base | Contributes to a late byte-write address; no initialization is shown here. |
| Address 0x087120b4 | Holds the 32-bit **primary pointer slot**. Its contents are captured as the **primary base**. |
| Address 0x08712120 | Holds the distinct 32-bit **table pointer slot**. Its contents are captured as the **table base**. |
| Other reachable memory | Supplies selectors, pointers, scalar values, and write destinations. |
| Secondary call-result channels | Supply a byte and address components separately from displayed ordinary results. |

The two pointer slots have established storage identities but unknown contents. They are not interchangeable.

Before the final call of each iteration, the displayed link-like value is assigned 0x08712031. Consequently, the next iteration’s displayed first-call target is 0x0871302f. This describes the visible dataflow, not a guarantee about preservation of implicit machine state by unknown callees.

### Widths and arithmetic

- Byte accesses are 8 bits.
- Halfword accesses are 16 bits.
- Word accesses are 32 bits.
- Paired results are 64 bits, separated into low and high 32-bit components.
- Unsigned halfword reads are zero-extended when consumed as wider values.
- Byte and halfword writes retain the low 8 or 16 bits, respectively.
- The first retained low word’s right shift is logical, yielding a value from 0 through 63.

Addresses and pointer-like values are represented through 32-bit quantities. Some additions, left shifts, and multiplications use signed 32-bit intermediates. The evidence does not establish language-level overflow behavior, address wrapping, alignment rules, or byte order. Where examples use truncating 32-bit arithmetic, that interpretation is explicitly conditional.

### Preconditions for uninterrupted execution

No validation is visible. Progress requires callable indirect targets, accessible memory of the stated widths, usable secondary results, an appropriate execution context, and continuation after the breakpoint.

These are operational requirements, not checked conditions. No failure return or recovery contract is established.

## Behavior

F-01 through F-37 identify call occurrences in execution order. Some occurrences have statically identical targets, as specified under **Dependencies**.

### Initial calls and primary updates

At iteration entry, a retained byte is initialized to zero. F-01 is invoked with two explicit zero arguments. Its displayed target is the current link-like value plus 0xffe.

After F-01, a separate zero byte is established for the later paired-component sum. F-02 and F-03 follow without displayed explicit arguments.

The function then reads the word at 0x087120b4 and captures it as the primary base. F-04 follows that read.

F-05 supplies a paired result. F-06 receives its low 32-bit component and the sum of its high and low 32-bit components, with the separately established zero byte added. Under truncating 32-bit arithmetic, the second argument is the low 32 bits of the component sum. Signed-overflow interpretation remains unresolved.

After F-06, a secondary 8-bit result is written at primary base + 0x11. F-07 then runs, followed by a zero-byte write at primary base + 0x15.

F-08 supplies an ordinary 16-bit result and a secondary pointer-like result. The ordinary result is written as a halfword at secondary pointer + 0x0c.

Both primary writes use the address captured before F-04. Changing the contents of 0x087120b4 after that capture does not, by itself, redirect those writes.

### First paired result and captured-object update

F-09 supplies a paired result. Its low component becomes the **first retained word**; its high component is used as an address.

Before another dependency call, the function performs two writes in order:

1. A zero halfword at high-component address + 0x2c.
2. A word at frame base + 0x26c containing the first retained word logically shifted right by 26 bits.

F-10 follows. The function then reads and captures a word-sized pointer from frame base + 0x320 as the **captured object base**.

F-11 and F-12 run before a word write at captured object base + 0x54. The value written is the first retained word shifted left by two bits as an unsigned 32-bit quantity, retaining its low 32 bits.

The captured object pointer is not visibly reloaded after F-11 or F-12. F-13, F-14, F-15, F-16, and F-17 follow in that order.

### Fixed-address selection and breakpoint

The function reads an unsigned halfword at 0x21c48994. Its zero-extended value becomes the **first selected base**. This value itself is used as an address; no intervening pointer-table lookup is shown.

F-18 runs. The function then reads a word at frame base + 0x280 and supplies it as the displayed argument to F-19.

Immediately after F-19 returns, execution encounters a software breakpoint with immediate 0x92.

All subsequent behavior is conditional on continuation beyond that breakpoint. No handler, recovery policy, or continuation guarantee is supplied.

After continuation, F-19’s result is shifted left by two bits for a later table-write offset. A separate constant-derived quantity equals 0x21c4. The function reads a word at first selected base + 0x44 and captures it as the **destination base**.

### Table update and second selection

F-20 runs, followed by F-21, which supplies a word result. The function reads a word at first selected base + 0x04. F-22 receives the F-21 result and that memory value as its two displayed arguments.

The word at 0x08712120 is then read and captured as the table base. F-23 runs before a halfword write at:

**table base + the shifted F-19 result**

The stored value is the low 16 bits of the captured table base. It is not the index, the first selected base, or a value fetched from the selected record.

F-24 follows. The function then reads a word-sized pointer at 0x2218, which is 0x21c4 + 0x54. This becomes the **second selected base**.

F-25 returns a pointer-like value. A halfword containing 0x21c4 is written at that address + 0x04.

F-26 and F-27 run. The function then reads an unsigned halfword at second selected base + 0x34 and zero-extends it as the **selector**.

### Second paired result and frame adjustment

F-28 supplies another paired result, separated into the **second low word** and **second high word**.

Before the next dependency call, the visible effects and calculations occur in this order:

1. Write the second low word at the current frame base + 0x258.
2. Read an unsigned halfword at second high word + 0x14, retaining it as the **copied halfword**.
3. Form a **low-word offset** by shifting the second low word left by six bits.
4. Form a **scaled high word** by multiplying the second high word by four.
5. Decrease the frame-like base by 0xf4.
6. Form a **selector offset** equal to four times the zero-extended selector.
7. Write the second low word at selector offset + 0x74.

The selector-derived write has no selected-object or table base added. Since the selector is an unsigned halfword, its four-times offset ranges from 0 through 0x3fffc.

The frame-relative +0x258 write precedes the frame adjustment. No corresponding restoration is visible before repetition. Whether this adjustment represents a literal machine-register change or an artifact of the supplied representation remains documented_only.

### Final destination updates and repetition

F-29 runs. The copied halfword is written as a zero-extended word at destination base + 0x54.

F-30 runs. The same captured value is written as a word at destination base + 0x64.

F-31 runs. A secondary pointer-like result from that call is added to the low-word offset. At the resulting address, the function writes the low 16 bits of the destination base.

F-32 and F-33 run. The function reads a byte at selector + selector offset, equivalently five times the selector. No additional base is shown.

F-34 runs. The captured byte is then written at:

**copied halfword + incoming register-held base + 0x19**

The source byte was captured before F-34. The register-held address contribution has no visible initialization in this function.

F-35 runs. The scaled high word is written as a word at destination base + 0x44.

F-36 runs. The link-like value is explicitly assigned 0x08712031. The function then reads a word at scaled high word + 0x44 and passes it as the displayed argument to F-37.

If F-37 returns and execution otherwise continues, the loop repeats unconditionally. The displayed next first-call target is then 0x08712031 + 0xffe, or 0x0871302f.

## Side effects

### Direct effects

An iteration reaching its end contains:

- 37 indirect-call occurrences.
- Three byte writes.
- Five halfword writes.
- Seven word writes.
- Byte, halfword, and word reads.
- One software breakpoint.
- A frame-like base decrease of 0xf4.
- Assignment of the link-like value used in the next displayed first-call target calculation.

Effects are incremental. No transaction, rollback, cleanup, or failure branch is visible.

### Ordering and aliasing

The access and call ordering above is observable behavior. Calls can observe prior writes or affect memory read afterward.

No non-aliasing guarantees are established. Fixed storage, captured objects, calculated destinations, and returned addresses may overlap. A later read may therefore observe an earlier write, subject to intervening dependency effects.

A captured scalar or address is reused where specified. This does not establish preservation of global memory, pointed-to contents, or unrelated implicit state.

### Ownership and interruption

No direct allocation, release, ownership transfer, or lifetime extension is shown. Pointer-like results are used without visible validation. Dependencies may manage memory, but their ownership and lifetime effects are unknown.

No status result is tested. An invalid access, unsuitable call target, nonreturning dependency, or breakpoint may prevent further progress. No recovery behavior is established.

## Dependencies

Dependency implementations and true signatures are unavailable. Call-occurrence labels do not imply distinct target identities.

### Statically established target equalities

| Call occurrences | Established relationship |
|---|---|
| F-05 and F-06 | Identical displayed target |
| F-09 and F-13 | Identical displayed target |
| F-17 and F-24 | Identical displayed target |
| F-19 and F-31 | Identical displayed target |
| F-25 and F-32 | Identical displayed target |

These equalities follow from the displayed target calculations, not from assumed functional similarity. They remain relevant even where the two occurrences have different displayed arguments or consume different result channels. A shared dependency may retain state between occurrences.

No additional target equality is asserted here. In particular, the incoming first-call target could coincide with another target depending on the incoming link-like value.

### Displayed interfaces

| Opaque call IDs | Visible interface or role |
|---|---|
| F-01 | Target is current link-like value + 0xffe; receives two zeros. |
| F-02–F-04 | No displayed explicit arguments. |
| F-05 | Supplies a paired result. |
| F-06 | Receives paired low word and component sum; supplies a secondary byte. |
| F-07 | No displayed explicit arguments. |
| F-08 | Supplies a halfword and secondary pointer-like result. |
| F-09 | Supplies the first retained paired result. |
| F-10–F-18 | Interleaved with the accesses specified above; no displayed explicit arguments. |
| F-19 | Receives a frame-relative word; supplies the table-indexing result. |
| F-20 | No displayed explicit arguments. |
| F-21 | Supplies a word result. |
| F-22 | Receives the F-21 result and a selected-memory word. |
| F-23–F-24 | Calls surrounding the table write and subsequent pointer read. |
| F-25 | Supplies a pointer-like result for a halfword write. |
| F-26–F-27 | Precede the selector read. |
| F-28 | Supplies the second paired result. |
| F-29–F-30 | Separate the copied-halfword word writes. |
| F-31 | Supplies a secondary address component. |
| F-32–F-36 | Interleaved with final byte and word accesses. |
| F-37 | Receives the word read at scaled high word + 0x44. |

“No displayed explicit arguments” does not mean no inputs. Registers, globals, memory, and calling-convention state may carry additional inputs. Unused ordinary results do not imply absence of outputs or side effects.

## Acceptance examples

These are conditional static acceptance cases, not executed tests.

1. **First-call targeting.** If the incoming link-like value is 0x1000 and the addition is representable, the first displayed target is 0x1ffe. After a completed iteration, the next displayed target is 0x0871302f, conditional on continued execution and the displayed implicit-state interpretation.

2. **Distinct pointer-slot bindings.** If 0x087120b4 contains one address and 0x08712120 contains another, the primary writes use the former captured address; the indexed table write uses the latter. Neither slot substitutes for the other.

3. **Shared dependency state.** F-05 and F-06 invoke the same target. A dependency maintaining state across those calls need not behave as though F-06 were an independent implementation. The same requirement applies to each equality pair listed above.

4. **Logical extraction.** If F-09’s low word is 0xffffffff, the frame-relative +0x26c write is 63, and the later captured-object +0x54 write is 0xfffffffc.

5. **Captured object pointer.** If the pointer captured after F-10 identifies one object and F-11 changes the source slot to identify another, the later +0x54 write still uses the captured address.

6. **Unsigned selector addressing.** For selector 3, the selector-derived word write targets 0x80 and the later byte read targets 0x0f. No table base is implicitly added.

7. **Store truncation and extension.** A destination base of 0x12345678 contributes 0x5678 to the halfword write after F-31. A copied halfword of 0xffff contributes 0x0000ffff to both destination word writes at +0x54 and +0x64.

8. **Breakpoint interruption.** If execution does not resume after the breakpoint, no later access or call is required to occur. Earlier effects have no visible rollback.

9. **Conditional repetition.** If F-37 returns, control repeats unconditionally. If it does not return, no alternate continuation is shown.

## Uncertainties

The following remain **documented_only**:

- Application purpose and field meanings.
- Dependency implementations, true signatures, hidden inputs, and side effects.
- Interpretation of secondary result channels.
- Preservation of globals and implicit machine state across calls.
- Address validity, lifetime, ownership, and aliasing.
- Literal runtime meaning of the frame-like base and its adjustment.
- Signed-overflow and signed-left-shift behavior for affected calculations.
- Address wrapping, alignment, byte order, and memory-mapped effects.
- Breakpoint handling and continuation policy.
- Reachability of fixed and unusually low addresses.

These unknowns do not erase established storage bindings, target equalities, or operand relationships. They also do not justify adding validation, hidden address bases, ownership rules, or recovery behavior.

## Evidence

This contract is based solely on static inspection of the supplied function representation.

Established evidence includes access widths, ordered calls and accesses, fixed storage addresses, captured-value reuse, paired-result separation, explicit zero extension and store truncation, breakpoint placement, target equalities, the first-call target formula, and unconditional repetition.

The constant-derived values 0x21c4, 0x2218, and 0x0871302f follow from the displayed arithmetic. Application intent and runtime interpretation are separately qualified.

No dependency implementations, runtime traces, memory maps, or executable validation were available. No runtime tests have been performed.
