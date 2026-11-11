<!-- Trusted provenance: F-84afe689f0fe; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-84afe689f0fe

## Purpose

F-84afe689f0fe repeatedly performs indirect calls, reads and writes memory of several widths, and executes a software breakpoint. The supplied representation contains no normal return, exit condition, or error-handling branch.

An application-level purpose is not established. The accesses may represent state transfer or initialization, but the evidence does not justify assigning device, queue, allocation, or synchronization semantics.

This contract describes the represented sequence, conditional on execution reaching each operation. It does not establish that the addresses are valid, that dependencies return, or that execution continues after the breakpoint.

## Interface

### Explicit interface

- No explicit parameters.
- No explicit return value.
- No represented normal return.
- An unconditional repetition boundary after the final indirect call.

### Implicit inputs

The function uses state outside its explicit interface:

| Input | Role |
|---|---|
| Incoming continuation-address state | Determines the first indirect-call target on the first iteration. |
| An implicit address base | Supplies state-relative addresses and is explicitly decreased during each iteration that reaches the update. |
| An incoming auxiliary register value | Contributes to the destination of a late byte store. |
| Global cell G1 | Supplies an early object address. |
| Global cell G2 | Supplies an indexed-store base. |
| Memory reached through these values and call results | Supplies further addresses, arguments, halfwords, and a byte. |
| Additional output-register state | Supplies one byte value and two address bases without established conventional return interfaces. |

G1 and G2 are descriptive labels for distinct global cells. Descriptive value names below are not source identifiers.

### Widths and arithmetic

The represented memory accesses have widths of 8, 16, or 32 bits. Address-bearing scalar values are represented as 32 bits.

**Three 64-bit call results are consumed:** those from F-000000000005, F-000000000009, and F-000000000028. Each is separated into low and high 32-bit halves. The result of each occurrence is independent of the other two.

For the machine-width interpretation used here:

- 32-bit arithmetic retains the low 32 bits.
- A 16-bit store retains the low 16 bits.
- An 8-bit store retains the low 8 bits.
- Unsigned halfword reads are zero-extended when used in wider calculations.
- The specified right shift of a low result half is logical.

These rules describe the represented machine operations, not portable signed-arithmetic semantics. Endianness, alignment requirements, address-space behavior, and the validity of treating narrow integers as addresses remain unresolved.

### Dependency naming

F-000000000001 through F-000000000037 identify **call occurrences in execution order within one iteration**. They do not imply 37 distinct targets. Known target equalities are listed under Dependencies.

A call described without explicit arguments has no arguments represented at that occurrence; this does not establish that it consumes no implicit inputs.

## Behavior

### Initial calls and early stores

An iteration begins by establishing a zero byte for an early store, then calling F-000000000001 with two explicit zero arguments. Its first-iteration target depends on incoming continuation-address state. A separate byte-sized arithmetic contribution is then established as zero.

The subsequent order is:

1. Call F-000000000002.
2. Call F-000000000003.
3. Read G1 and capture its 32-bit value as **early object**.
4. Call F-000000000004.
5. Call F-000000000005 and capture the first 64-bit result.

Call the halves of that result **first low** and **first high**. F-000000000006 receives two explicit 32-bit arguments: first low, followed by first high plus first low plus the zero arithmetic contribution, retaining 32 bits.

After that call, one byte is stored at early object + 17. The byte comes from additional output-register state associated with the call, rather than an established ordinary return value.

The function then:

1. Calls F-000000000007.
2. Stores the previously established zero byte at early object + 21.
3. Calls F-000000000008, consuming a 16-bit result.
4. Stores that result at an additional output-register-supplied base + 12.

The early object value was captured before F-000000000004. A later change to G1 does not change the represented address operands for either early-object store. This does not establish that the captured storage remains valid.

### Second composite result and state-relative writes

F-000000000009 supplies the second 64-bit result. Denote its halves **second low** and **second high**.

The ordered effects are:

1. Store a zero halfword at second high + 44.
2. Store second low logically shifted right by 26 as a 32-bit word at implicit base + 620.
3. Call F-000000000010.
4. Read a 32-bit destination base from implicit base + 800.
5. Call F-000000000011.
6. Call F-000000000012.
7. Store second low shifted left by two, retaining 32 bits, at the captured destination base + 84.
8. Call F-000000000013.
9. Call F-000000000014.
10. Call F-000000000015.
11. Call F-000000000016.
12. Call F-000000000017.

The value written at implicit base + 620 is between 0 and 63. Both shifts use the captured low half from F-000000000009, not the earlier result from F-000000000005.

The destination base is captured after F-000000000010 and before F-000000000011. Subsequent changes to the cell from which it was read do not redirect the represented store.

### Narrow address input and breakpoint

The function reads an unsigned 16-bit value from absolute address 0x21C48994. Denote the zero-extended result **narrow base**.

It then:

1. Calls F-000000000018.
2. Reads a 32-bit argument from implicit base + 640.
3. Calls F-000000000019 with that argument and captures its 32-bit result as **selection**.
4. Executes software breakpoint number 0x92.

All remaining behavior in the iteration requires execution to continue after this breakpoint.

After continuation, selection is shifted left by two, retaining 32 bits, to form **selection offset**. Constant arithmetic establishes **fixed base** as 0x21C4. A 32-bit read at narrow base + 68 supplies **linked base**.

Narrow base is used directly as a zero-extended 16-bit address component. There is no represented reconstruction of upper address bits.

### Indexed halfword store and secondary address capture

After the linked-base read, the order is:

1. Call F-000000000020.
2. Call F-000000000021 and retain its 32-bit result.
3. Read a 32-bit value at narrow base + 4.
4. Call F-000000000022 with the retained result as its first explicit argument and the memory value as its second.
5. Read G2 and capture its 32-bit value as **table base**.
6. Call F-000000000023.
7. Store a halfword at table base + selection offset, using the low 16 bits of table base as the stored value.
8. Call F-000000000024.
9. Read a 32-bit value at fixed base + 84, absolute address 0x2218, as **secondary base**.
10. Call F-000000000025 and consume a 32-bit address-bearing result.
11. Store halfword 0x21C4 at that result + 4.
12. Call F-000000000026.
13. Call F-000000000027.

Selection controls the indexed destination, not its stored value. The value comes from table base itself.

Table base is captured before F-000000000023. Changes to G2 during that call do not redirect the represented halfword store. No bounds check accompanies selection-based addressing.

### Third composite result and implicit-base update

The function reads an unsigned halfword at secondary base + 52 and zero-extends it as **index**. It then calls F-000000000028, consuming the third 64-bit result. Denote its halves **third low** and **third high**.

The following operations occur in order:

1. Store third low as a 32-bit word at implicit base + 600.
2. Read an unsigned halfword at third high + 20 and zero-extend it as **small value**.
3. Form **shifted low** from third low shifted left by six, retaining 32 bits.
4. Form **scaled high** from third high multiplied by four, retaining 32 bits.
5. Decrease the implicit base by 244.
6. Form **index offset** as index multiplied by four.
7. Store third low as a 32-bit word at index offset + 116.

The state-relative store uses the implicit base before its explicit decrement. No matching restoration is represented at the repetition boundary.

Index is between 0 and 65535, so index offset is between 0 and 262140. Neither its multiplication by four nor the subsequent addition of 116 exceeds 32 bits.

### Final stores and byte transfer

The remaining sequence is:

1. Call F-000000000029.
2. Store small value as a 32-bit word at linked base + 84.
3. Call F-000000000030.
4. Store small value as a 32-bit word at linked base + 100.
5. Call F-000000000031.
6. Store the low 16 bits of linked base at shifted low plus an additional output-register-supplied base.
7. Call F-000000000032.
8. Call F-000000000033.
9. Read one byte at index + index offset, equivalent to five times index.
10. Call F-000000000034.
11. Store the captured byte at small value + incoming auxiliary register value + 25.
12. Call F-000000000035.
13. Store scaled high as a 32-bit word at linked base + 68.
14. Call F-000000000036.
15. Establish a fixed continuation-address value.
16. Read a 32-bit value at scaled high + 68.
17. Call F-000000000037 with that value.

The two small-value word stores are separated by F-000000000030. They are not an atomic pair. Because small value is zero-extended, each writes a value between 0 and 65535.

The late byte store uses the byte captured before F-000000000034. It does not reread the source afterward.

The fixed continuation-address assignment determines the represented target of F-000000000001 on the next iteration. The original incoming continuation state is therefore relevant only to the first iteration’s first call.

After F-000000000037, repetition is unconditional.

## Side effects

### Memory and execution state

Direct effects include byte, halfword, and word stores; reads from fixed and computed addresses; one explicit implicit-base decrease per iteration reaching that point; and a software breakpoint.

No non-aliasing guarantee is established. A destination may overlap another destination, a later input location, or a global cell. Access order therefore matters. Widths must also be preserved: a halfword store is not interchangeable with a word store.

The explicit base update decreases it by 244. Across completed iterations, the representation supplies no compensating increase. This identifies the function’s explicit update, not a general guarantee that unknown dependencies preserve architectural state.

### Dependency effects and captured values

No dependency is established as pure, nonblocking, nonthrowing, or preserving global memory. A call could change later-read memory or invalidate storage reached through an earlier captured address.

Captured operands remain the operands used by the supplied representation. That observation does not establish a complete machine-level register-preservation convention.

### Ownership and failures

No allocation, release, reference-count update, or ownership transfer is explicitly identified. Passing an address to a dependency or storing it does not establish ownership transfer.

There are no represented null checks, bounds checks, alignment checks, status checks, retries, or rollback operations. Invalid memory, invalid indirect-call targets, nonreturning dependencies, or breakpoint handling may prevent further progress. Environment-specific failure behavior is unresolved. No explicit operation undoes earlier completed effects.

## Dependencies

The behavior section identifies all 37 ordered call occurrences.

The following pairs have equal represented targets after constant arithmetic:

| Call occurrences | Established relationship |
|---|---|
| F-000000000005 and F-000000000006 | Same target; the latter receives two explicit arguments. |
| F-000000000009 and F-000000000013 | Same target; only the former supplies a consumed 64-bit result. |
| F-000000000017 and F-000000000024 | Same target. |
| F-000000000019 and F-000000000031 | Same target; the former receives an explicit argument and supplies the consumed selection result. |
| F-000000000025 and F-000000000032 | Same target; only the former supplies a consumed address-bearing result. |

These equalities do not establish reliable prototypes. Differing represented argument lists do not prove overloads, ignored arguments, or independent implementations.

F-000000000001 is exceptional: its target initially depends on incoming continuation state and subsequently on the fixed continuation value established near the preceding iteration’s end.

The three consumed 64-bit results belong to F-000000000005, F-000000000009, and F-000000000028. The 16-bit result belongs to F-000000000008. Additional output-register values used after F-000000000006, F-000000000008, and F-000000000031 have no established conventional interface.

Dependency internals, ownership behavior, error protocols, and preservation guarantees remain unknown.

## Acceptance examples

These are conditional static acceptance examples, not executed tests. Each assumes the required memory accesses and calls complete, and that the breakpoint resumes when later behavior is considered.

1. **First composite-result addition wraps.**  
   If first low is 0xFFFFFFFF and first high is 2, F-000000000006 receives 0xFFFFFFFF and 1. The separate zero contribution does not change the sum.

2. **Second-result shifts retain their distinct meanings.**  
   If second low is 0xFC000001, the word at implicit base + 620 receives 63. The later store at the captured destination base + 84 receives 0xF0000004.

3. **The third result supplies independent values.**  
   If third low is 0x40000001 and third high is 0x40000002, the state-relative word store receives 0x40000001, shifted low is 0x00000040, and scaled high is 0x00000008. The small-value halfword read occurs at 0x40000016.

4. **A narrow value remains a direct address component.**  
   If narrow base is 0x1234, linked base is read at 0x1278, and the later argument for F-000000000022 is read at 0x1238. No upper bits are added.

5. **Indexed destination and stored value have different sources.**  
   If selection is 3 and table base is 0x12345678, the halfword destination is 0x12345684 and the stored value is 0x5678.

6. **Breakpoint noncontinuation prevents later operations.**  
   If breakpoint 0x92 does not resume, the table store, third-result processing, explicit base decrement, and late byte transfer are not reached. Earlier effects are not rolled back.

7. **The state-relative store precedes the decrement.**  
   If the implicit base is 0x1000 at the third-low word store, that store targets 0x1258. The subsequent explicit decrement produces 0x0F0C.

8. **Maximum index arithmetic stays within 32 bits.**  
   If index is 65535, index offset is 262140, the third-low word destination is 262256, and the byte-source address is 327675.

9. **Small value is zero-extended.**  
   If small value is 0xFFFF, both linked-base word stores write 0x0000FFFF, not 0xFFFFFFFF.

10. **Intervening calls preserve observable separation.**  
    F-000000000030 is reached after the store at linked base + 84 and before the store at linked base + 100. F-000000000034 is reached after the byte read and before its destination write. Those calls may observe or alter memory; the later byte write still uses the captured byte.

## Uncertainties

The following remain **documented_only**:

- Application-level meaning of the loop, global cells, fixed addresses, and offsets.
- Actual calling conventions, including implicit arguments and additional output-register values.
- Architectural identity and validity of the implicit address base.
- Whether the unconditional repetition and unbalanced base update accurately represent intended execution.
- Valid memory ranges, alignment rules, endianness, and memory-mapped effects.
- Runtime validity of narrow integers used directly as addresses.
- Whether dependencies return, preserve relevant state, release storage, or invalidate captured addresses.
- Breakpoint handling, resumption, and any resulting state changes.
- Reliability of recovered signedness and callable signatures.

No inferred intent resolves these unknowns. Repeated stores do not establish synchronization, and indexed addressing does not establish an allocated array or bounds guarantee.

## Evidence

**Established static evidence:** the supplied representation contains an unconditional loop, 37 indirect call occurrences per fully reached iteration, one software breakpoint, byte/halfword/word accesses, three consumed 64-bit results, an explicit implicit-base decrease of 244, and no represented normal return or error branch.

**Arithmetic deductions:** the listed target equalities, fixed base 0x21C4, absolute read addresses 0x21C48994 and 0x2218, halfword-derived index bounds, and truncation examples follow from the supplied expressions under the stated machine-width interpretation.

**Inferred intent:** state transfer or publication is plausible but is not an established purpose or guarantee.

**Runtime evidence:** none. No runtime tests have been performed.
