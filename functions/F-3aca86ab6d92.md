<!-- Trusted provenance: F-3aca86ab6d92; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-3aca86ab6d92

## Purpose

F-3aca86ab6d92 repeatedly performs an ordered sequence of indirect calls, memory accesses, and state updates. Each iteration includes a software breakpoint with immediate value 0x92. Operations after that breakpoint are conditional on execution continuing.

The supplied static representation establishes state transformations, not an application-level purpose. It does not establish that the function initializes a device, processes a particular record format, or implements a conventional service.

There is no represented normal return. If dependencies return, memory accesses succeed, and the execution environment resumes past each breakpoint, the sequence repeats indefinitely.

## Interface

### Explicit inputs and outputs

- No explicit parameters are represented.
- No explicit return value or normal return path is represented.
- Outputs consist of memory writes, dependency invocations, the breakpoint, and updates to represented execution state.

### Implicit inputs

The function uses:

- An incoming **link-related value** to determine the first indirect call target.
- An **inherited register value** in a late byte-store destination.
- A **working base** used for offset-addressed accesses and decreased during each iteration.
- Two word-sized global values, called **global base A** and **global base B**.
- Memory reached through these values, fixed addresses, selectors, and dependency results.
- Secondary register outputs from certain dependencies.

These names describe data roles only. They do not establish object types, register-preservation rules, or ownership.

### Widths and layouts

The represented accesses use:

| Quantity | Width and interpretation |
|---|---|
| Byte | 8 bits |
| Halfword | 16 bits |
| Word | 32 bits |
| Paired result | 64 bits, divided into low and high 32-bit words |
| Selector | Unsigned halfword, zero-extended for subsequent calculations |

Halfword stores retain the low 16 bits of their source. Word stores retain 32 bits. Left shifts and word-sized products are described as 32-bit bit-vector calculations, discarding bits above bit 31.

Address additions are described using the represented 32-bit address arithmetic. This does not establish the behavior of invalid addresses, alignment faults, or arithmetic in any higher-level implementation language.

No packed structure layout is established. Offsets identify individual accesses; they do not prove that the surrounding memory belongs to a single object. Byte order is unspecified.

### Preconditions for uninterrupted traversal

A complete iteration requires callable indirect targets, returning dependencies, accessible memory at every indicated width, usable inherited and secondary register values, and continuation past the breakpoint.

No represented validation enforces these conditions. In particular, there are no bounds checks, null checks, alignment checks, or checks on selector-derived addresses.

## Behavior

The labels F-D01 through F-D37 identify call sites in execution order within one iteration. They are opaque reference IDs, not verified dependency identities. Different labels may resolve to the same target.

A call with “no explicit arguments” has no arguments shown in the supplied representation. Its actual interface may still depend on registers or memory.

### 1. Opening calls and byte writes

At the start of each iteration:

1. Establish a zero byte for a later store.
2. Invoke F-D01 with two zero-valued word arguments. Its target is the current link-related value plus 0xFFE.
3. Establish another zero byte, subsequently used in the F-D06 argument calculation.
4. Invoke F-D02 and F-D03, in that order.
5. Read global base A as a word and retain that captured base.
6. Invoke F-D04.
7. Invoke F-D05 and divide its paired result into low and high words.
8. Invoke F-D06 with the low word as its first argument and the word-width sum of the high word, low word, and second zero byte as its second argument.
9. Store the secondary byte output exposed by F-D06 at captured base A plus 0x11.
10. Invoke F-D07.
11. Store the first zero byte at captured base A plus 0x15.

The two byte destinations use the same captured base. There is no represented reread of global base A between them. A dependency changing the global slot therefore does not, by itself, change the base selected by the shown data flow.

### 2. Initial halfword and word writes

Continue as follows:

1. Invoke F-D08.
2. Store its halfword result at the secondary word-sized base exposed by that call, plus 0x0C.
3. Invoke F-D09 and divide its paired result into low and high words.
4. Store a zero halfword at the high-word address plus 0x2C.
5. Store the low word logically shifted right by 26 at working base plus 0x26C.
6. Establish the constant 0x21C48960.
7. Invoke F-D10.
8. Read and retain a word-sized destination base from working base plus 0x320.
9. Invoke F-D11, then F-D12.
10. Store the low word from F-D09 shifted left by two at the captured destination base plus 0x54.

The right-shifted value lies between 0 and 63. The left-shifted value retains only 32 bits.

The halfword zero write precedes the working-base-relative word write. The pointer read at offset 0x320 occurs after F-D10 and before F-D11 and F-D12. The later store uses that captured pointer, rather than rereading the working-base-relative slot.

### 3. First selector and breakpoint

The next operations are:

1. Invoke F-D13, F-D14, F-D15, F-D16, and F-D17, in order.
2. Read an unsigned halfword at 0x21C48994, which is 0x21C48960 plus 0x34. Its zero-extended value is the **first selector**.
3. Invoke F-D18.
4. Read a word at working base plus 0x280.
5. Pass that word as the explicit argument to F-D19 and retain its word-sized result.
6. Execute the software breakpoint with immediate value 0x92.

No post-breakpoint operation is required unless execution continues. The representation does not establish whether the breakpoint halts execution, invokes a handler, or permits resumption.

### 4. Post-breakpoint indexed operations

After continuation:

1. Shift the F-D19 result left by two, retaining 32 bits. This is the **first displacement**.
2. Derive 0x21C4 by logically shifting 0x21C48960 right by 18 and multiplying by four.
3. Read a word at first selector plus 0x44. Retain it as the **record base**.
4. Invoke F-D20.
5. Invoke F-D21 and retain its word-sized result.
6. Read a word at first selector plus 0x04.
7. Invoke F-D22 with the F-D21 result as its first explicit argument and the just-read memory word as its second.
8. Read global base B as a word.
9. Invoke F-D23.
10. Store a halfword at captured base B plus the first displacement. The stored value is the low 16 bits of captured base B.
11. Invoke F-D24.
12. Read a word at 0x2218, the sum of 0x21C4 and 0x54. Retain it as the **intermediate base**.
13. Invoke F-D25 and use its word-sized result as a destination base.
14. Store the halfword 0x21C4 at that destination base plus 0x04.
15. Invoke F-D26, then F-D27.

The first selector is used directly as an address base. No represented operation scales it or adds a separate table base before the reads at offsets 0x44 and 0x04.

The store through global base B uses the captured value despite the intervening F-D23 call. The intermediate-base read precedes F-D25 and the halfword store through F-D25’s result.

### 5. Selector capture, working-base movement, and record writes

Next:

1. Read an unsigned halfword at intermediate base plus 0x34. Its zero-extended value is the **second selector**.
2. Invoke F-D28 and divide its paired result into low and high words.
3. Store the low word at working base plus 0x258.
4. Read an unsigned halfword at the high-word address plus 0x14. Its zero-extended value is the **third selector**.
5. Derive a displacement from the low word shifted left by six, and a **scaled high word** from the high word multiplied by four. Both retain 32 bits.
6. Decrease the working base by 0xF4.
7. Form the second selector multiplied by four and store the low word from F-D28 at that displacement plus 0x74.
8. Invoke F-D29.
9. Store the third selector as a full word at record base plus 0x54.
10. Invoke F-D30.
11. Store the third selector as a full word at record base plus 0x64.
12. Invoke F-D31.
13. Store the low 16 bits of the record base at the sum of the low-word-derived displacement and the secondary word-sized output exposed by F-D31.

The store at working base plus 0x258 uses the base before its decrement. Relative to the resulting base, that address is offset 0x34C.

The two third-selector stores are separate 32-bit writes. F-D30 occurs between them, so they are not an atomic pair and need not leave equal final contents if the dependency changes memory.

The indexed destination at 0x74 plus four times the second selector has no additional represented base.

### 6. Byte transfer, final state update, and repetition

The iteration finishes in this order:

1. Invoke F-D32, then F-D33.
2. Read a byte at five times the second selector: the selector plus its previously formed fourfold displacement.
3. Invoke F-D34.
4. Store the captured byte at third selector plus the inherited register value plus 0x19.
5. Invoke F-D35.
6. Store the scaled high word at record base plus 0x44.
7. Invoke F-D36.
8. Set the represented link-related value to 0x08712031.
9. Read a word at scaled high word plus 0x44.
10. Invoke F-D37 with that word as its explicit argument.
11. Begin another iteration without a represented restoration of the working base.

The link-related state update precedes the final argument read. If that read faults, the represented sequence has already reached the link-related update, but has not invoked F-D37. The function shows no rollback of that state update or earlier memory writes.

On the shown uninterrupted data flow, the next iteration’s F-D01 target is therefore 0x0871302F: 0x08712031 plus 0xFFE. This is not a guarantee that an unknown dependency or execution environment cannot alter relevant machine state.

The final read occurs after both the record-base-plus-0x44 store and F-D36. Its address is scaled high word plus 0x44, not necessarily record base plus 0x44. Aliasing or dependency writes can affect the value passed to F-D37.

## Side effects

A fully traversed iteration contains:

- Thirty-seven indirect dependency calls.
- One software breakpoint.
- Three byte stores.
- Five halfword stores.
- Seven word stores.
- A working-base decrement of 0xF4.
- A late link-related state update before the final memory read and call.

These are direct effects visible in the supplied representation. Dependencies may have additional effects.

The memory accesses are ordered as described. No synchronization, barrier, atomic transaction, or rollback mechanism is represented. Aliasing can make an earlier store affect a later read, including accesses that appear to have unrelated descriptive names.

The function shows no direct allocation, release, ownership transfer, or lifetime management. Pointer-shaped values establish neither ownership nor permission to free memory. No dependency is assumed to preserve globals, preserve arbitrary registers, allocate memory, or release memory.

There is no represented error result, recovery branch, retry policy, or cleanup path. A nonreturning dependency, faulting access, or nonresuming breakpoint can interrupt execution after partial mutation. The consequences of the interruption belong to the execution environment and are not established here.

## Dependencies

| Opaque call-site IDs | Established data role |
|---|---|
| F-D01 | Receives two zero words; target depends on the current link-related value. |
| F-D02–F-D04 | Ordered opening calls around capture of global base A. |
| F-D05 | Supplies the paired result used to construct F-D06’s arguments. |
| F-D06 | Receives two words and exposes the secondary byte used in the first byte store. |
| F-D07 | Separates the initial byte stores. |
| F-D08 | Supplies a halfword result and secondary destination base. |
| F-D09 | Supplies a paired result for the zero-halfword destination and shifted word writes. |
| F-D10–F-D18 | Ordered calls surrounding pointer capture, a shifted store, and the first-selector read. |
| F-D19 | Receives the word read at working base plus 0x280; supplies the first displacement’s input. |
| F-D20–F-D23 | Surround the second first-selector-based read, F-D22’s two-argument call, and capture of global base B. |
| F-D24–F-D27 | Surround the fixed-address word read and the halfword store through F-D25’s result. |
| F-D28 | Supplies the paired result used in the final indexed, record, and pointer-related operations. |
| F-D29–F-D30 | Separate the indexed word store and the two third-selector word stores. |
| F-D31 | Exposes a secondary base used in a halfword destination. |
| F-D32–F-D34 | Precede and surround the final byte read before its destination store. |
| F-D35–F-D36 | Precede the final record-field store and the late link-update/read sequence, respectively. |
| F-D37 | Receives the final memory word after the link-related update, before repetition. |

No dependency implementation or complete external calling convention was supplied. Secondary outputs are represented data dependencies, not proof of a standard multi-result interface.

## Acceptance examples

These are conditional static acceptance criteria, not executed tests. Each assumes execution reaches the relevant operation and intervening effects remain consistent with the stated conditions.

1. **Bit extraction and truncation.** If F-D09’s low word is 0xFC000003, the word at working base plus 0x26C receives 63. The later shifted store through the captured destination receives 0xF000000C.

2. **Global-base halfword store.** If captured global base B is 0x12345678 and F-D19 returns 3, the halfword destination is 0x12345684 and the stored value is 0x5678.

3. **Unsigned first selector.** A first selector of 0xFFFF produces read addresses 0x10043 and 0x10003. It is not interpreted as negative one.

4. **Second-selector indexing.** A second selector of 7 produces indexed word destination 0x90 and later byte-source address 0x23.

5. **Selector field widths.** A third selector of 0xFFFF produces full-word stores of 0x0000FFFF at record base plus 0x54 and plus 0x64.

6. **Breakpoint interruption.** If execution does not continue beyond breakpoint 0x92, the subsequent record-base read and all later operations in that iteration are not reached. No rollback is represented for earlier writes.

7. **Working-base progression.** If an iteration begins with working base B and reaches the decrement, the represented base becomes B minus 0xF4. The earlier store at B plus 0x258 is then at new-base offset 0x34C.

8. **Intervening mutation.** If F-D30 changes the word at record base plus 0x54, the subsequent store at plus 0x64 does not itself restore the earlier field.

9. **Final-read fault ordering.** If F-D36 returns and the read at scaled high word plus 0x44 faults, the represented link-related value has already been set to 0x08712031. F-D37 has not been invoked, and a new iteration has not begun.

10. **Successful repetition.** If the final read succeeds, F-D37 returns, and relevant state is not externally changed, the next F-D01 target is 0x0871302F. The working-base decrement is not undone by the represented body.

## Uncertainties

### Established static evidence

The supplied representation shows unconditional repetition, explicit call order, access widths, offsets, selector zero-extension, arithmetic transformations, breakpoint value, working-base decrement, and the link-related update before the final read.

### Inferred intent

“Selector,” “record base,” and “intermediate base” are descriptive terms. They suggest indexed or record-oriented manipulation but establish no application semantics.

The working-base decrement could reflect execution-frame movement, but identifying it as stack management would exceed the supplied evidence.

### documented_only

The following remain unresolved:

- Actual dependency identities, full interfaces, side effects, and termination behavior.
- Validity and lifetime of secondary register outputs and inherited values.
- Whether machine-level values survive intervening calls exactly as the supplied representation indicates.
- Initial working-base and link-related state requirements.
- Breakpoint handling, fault handling, and asynchronous state changes.
- Address-space mapping, alignment requirements, byte order, concurrency, and device-memory semantics.
- Whether low-address accesses are intentional and valid.
- Ownership, allocation, release, and synchronization obligations.
- Whether the supplied representation fully and accurately reconstructs executable behavior.

These unknowns do not justify assumptions of safe memory, preserved globals, conventional dependency interfaces, or fault recovery.

## Evidence

This contract uses only the supplied static representation of F-3aca86ab6d92. The prior draft and reviewer feedback are not independent evidence. No dependency implementations or runtime observations were supplied. No runtime tests have been performed.

Arithmetic checks from the supplied constants establish that 0x21C48960 shifted right by 18 and multiplied by four is 0x21C4; adding 0x54 gives 0x2218. Adding 0xFFE to the late link-related value 0x08712031 gives 0x0871302F.

Unsigned halfword selectors range from 0 through 65535, so their fourfold and fivefold products fit within 32 bits. Shifts and products derived from unrestricted words can discard high bits.

The final ordering is explicit in the supplied representation: F-D36 returns, the link-related value is established, the final argument word is read, and F-D37 is invoked. This supports the partial-state distinction when that read fails.
