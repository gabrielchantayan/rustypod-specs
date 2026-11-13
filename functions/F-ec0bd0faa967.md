<!-- Trusted provenance: F-ec0bd0faa967; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-ec0bd0faa967

## Purpose

F-ec0bd0faa967 performs an ordered sequence of indirect dependency calls, byte reads, byte writes, and 32-bit writes. Visible effects include clearing selected locations, storing scaled dependency results, extracting bit fields into bytes, and clearing the 32-bit location at absolute address 0x24 on four separate occasions.

No application-level purpose is established. Descriptions such as initialization, serialization, allocation, or device control would require evidence beyond the supplied representation.

The represented body ends at an invalid-instruction boundary, not a normal return. This contract describes the visible execution prefix, conditional on execution following that representation. It does not promise successful completion.

## Interface

### Explicit inputs

| Input | Represented width and interpretation | Visible role |
|---|---|---|
| First argument | Signed 32-bit integer | Supplies three identical left-shifted arguments and a fourth differently shifted argument to one dependency. |
| Second argument | 32 opaque bits | Forwarded to the first dependency. |
| Third argument | 32 opaque bits | No explicit consumption in the visible body. |
| Fourth argument | Unsigned 32-bit integer | Supplies a direct byte destination, scaled selector-read addresses, and masked values passed or stored later. |

The fourth argument is not established to designate a conventional object. Its uses combine numeric scaling, direct addressing, and masking.

The third argument’s lack of explicit use does not prove that its incoming machine state is irrelevant to dependencies with unknown calling conventions.

### Additional entry state

The represented body also consumes state outside those four arguments:

- A link-related value determines the first indirect call target.
- An unsigned 32-bit word supplies an early byte’s contents.
- An address supplies the base for the first selector-byte read.
- An auxiliary register value observed after a later call supplies arguments to another dependency.

The origin and caller obligations for this state are not established.

### Outputs and preconditions

No normal return or returned value is visible. The represented interface is void-like, but no complete machine-level output convention is established.

Every later event requires all preceding calls to return and preceding accesses to succeed. The environment must permit the reached indirect calls and the specified memory-access widths and alignments. No visible validation enforces these requirements.

The actual outcome at the invalid-instruction boundary is **documented_only**: a trap, stop, transfer of control, or disagreement with the recovered representation cannot be resolved here.

## Behavior

### Widths, arithmetic, and address notation

The following mathematical names describe contract values, not implementation identifiers:

- **P** is the unsigned fourth argument.
- **M** is P masked to its low 22 bits.
- **Q** is P multiplied by 1,024, reduced modulo 2³².
- **T** is P multiplied by 67,108,864, reduced modulo 2³².

The reductions defining Q and T are established by the represented unsigned 32-bit multiplication. They occur before assignment to the subsequently represented signed 32-bit holders. Overflow of these unsigned multiplications is therefore not an unresolved signed-overflow case.

Q depends only on P’s low 22 bits; T depends only on its low six bits. The direct calculation P + 0x0F is also unsigned 32-bit arithmetic and reduces modulo 2³².

Two masked values require no wrap:

- M × 128 is at most 0x1FFFFF80.
- M × 16 is at most 0x03FFFFF0.

Other arithmetic is not uniformly unsigned. The first argument, many dependency results, the auxiliary value, and the extracted halves of the 64-bit result are represented as signed 32-bit values. Their potentially overflowing multiplication, addition, and left shifts must not be assigned portable unsigned-wrap semantics without further evidence. Conversions of out-of-range unsigned values to signed holders, and conversion of numeric results to effective addresses, remain **documented_only**.

For the two explicitly unsigned late results, multiplication and left shift retain unsigned 32-bit behavior. In particular, the final selector-read calculation, eight times the later unsigned result plus one, reduces modulo 2³².

A selector read fetches exactly one unsigned byte. That byte is zero-extended before adding 0x11, giving a byte-write destination from 0x11 through 0x110. It is not a pointer-sized address loaded from memory.

Dependency IDs below identify call occurrences. Distinct IDs do not establish distinct underlying functions. “No explicit arguments” means only that the supplied representation shows none.

### Entry calls and first selector-dependent write

F-A01 receives the second argument. Its target is derived from incoming link-related state.

F-A02 is called with no explicit arguments, and its 32-bit result is passed to F-A03. F-A04, F-A05, and F-A06 follow with no explicit arguments. F-A07 then supplies a 32-bit result retained for F-A08.

After F-A07 returns, one byte is read at the hidden incoming address plus one. At the selected destination, selector + 0x11, the function writes bits 22 through 29 of the hidden unsigned word.

F-A08 receives the F-A07 result shifted left by two. F-A09 through F-A12 follow in order with no explicit arguments.

The selector is read after these preceding calls have executed, not captured at function entry.

### Direct byte clear and Q-based selection

Next, a byte is cleared at the address calculated from P + 0x0F. A selector byte is then read at Q + 1, and a byte is cleared at that selector + 0x11.

F-B01 receives four arguments: three copies of the first argument shifted left by 21, then the first argument shifted left by 23.

F-B02 supplies a 32-bit value to F-B03. F-B04 through F-B06 follow without explicit arguments. F-B07 supplies another 32-bit result.

After F-B07, the byte at Q + 1 is read again. A byte is cleared at this new selector + 0x11. This selector is retained across the following calls.

F-B08 receives the F-B07 result shifted left by two. F-B09 through F-B12 follow without explicit arguments.

The retained selector then determines a 32-bit zero write at selector + 0x44. No new selector read occurs for this word write. A separate 32-bit zero write at absolute address 0x24 follows.

### Further word clears and a split 64-bit result

F-C01 receives, in order, zero, 0x98, Q, and 0x98.

F-C02 supplies a value to F-C03. F-C04 supplies another value to F-C05. F-C06 through F-C08 follow without explicit arguments. F-C09 supplies a 32-bit result.

A 32-bit zero is written at four times the F-C09 result plus 0x44. The 32-bit location at address 0x24 is then cleared.

F-C10 receives four times the F-C09 result. F-C11 through F-C13 follow without explicit arguments. F-C14 supplies a represented 64-bit result.

F-C15 receives the low 32-bit half of that result, interpreted as represented signed 32-bit data and shifted left by two. After F-C15 returns, a 32-bit zero is written at twice the high 32-bit half of the earlier F-C14 result plus 0x44. That high half is also converted to the represented signed 32-bit form before scaling. Another 32-bit clear at address 0x24 follows.

F-C16 receives three explicit arguments: zero, an auxiliary register value observed after F-C15, and that auxiliary value shifted left by two.

The auxiliary value must not be substituted with an assumed ordinary return value. Neither its ABI meaning nor a standard two-result convention for F-C15 is established.

### Masked propagation and result-dependent clearing

F-D01 has no explicit arguments. F-D02 receives M × 128. F-D03 through F-D05 follow without explicit arguments. F-D06 supplies a 32-bit result.

A 32-bit zero is written at four times that result plus 0x44, followed by the fourth separate 32-bit clear at address 0x24. F-D07 then receives four times the same result.

F-D08 through F-D10 follow without explicit arguments. F-D11 supplies a result whose left shift by two is passed to F-D12.

The result returned by F-D12 determines a 32-bit zero write at four times that result plus 0x14. F-D13 then receives three copies of the F-D12 result, followed by M × 128.

F-D14 through F-D16 have no explicit arguments. F-D17 supplies a result. A 32-bit zero is written at four times that result plus 0x14, and F-D18 receives four times the same result.

F-D19 through F-D21 follow without explicit arguments. F-D22 supplies a result. Four times that result is stored as a 32-bit value at sixteen times that result plus 0x14.

### Scaled-result stores

F-E01 receives the scaled value just stored. F-E02 through F-E04 follow without explicit arguments. F-E05 supplies a new result.

A 32-bit store writes the earlier F-D22-derived scaled value at four times the F-E05 result plus 0x14. The new result determines the destination, not the stored contents.

F-E06 receives four times the F-E05 result. F-E07 through F-E09 follow without explicit arguments. F-E10 supplies another result.

Four times the F-E10 result is stored as a 32-bit value at sixteen times that same result plus 0x14. F-E11 receives four times the result and sixteen times the result.

F-E12 supplies a value to F-E13. F-E13’s second explicit argument is M × 16.

F-E14 through F-E16 follow without explicit arguments. F-E17 supplies a result. A 32-bit store writes M × 128 at four times the F-E17 result plus 0x14.

F-E18 receives four times the F-E17 result. F-E19 through F-E21 follow without explicit arguments. F-E22 supplies an unsigned 32-bit result used in subsequent byte writes.

### T-based byte writes

After F-E22, a selector byte is read at T + 1. At selector + 0x11, the function writes the low eight bits of four times the F-E22 result.

F-F01 receives the F-E22 result shifted left by two. F-F02 supplies a value to F-F03. F-F04 through F-F06 follow without explicit arguments. F-F07 supplies a result.

The selector byte at T + 1 is read again. At this newly selected destination + 0x11, the function writes bits 20 through 27 of the earlier F-E22 result. The represented mask to 30 bits does not change those eight output bits.

F-F08 receives the F-F07 result shifted left by two. F-F09 through F-F11 follow without explicit arguments. F-F12 supplies a new unsigned 32-bit result.

A third selector byte is read at T + 1 and retained. At retained selector + 0x11, the function writes the low eight bits of four times the F-F12 result. The represented narrowing before multiplication does not change these final eight stored bits.

F-F13 receives the retained selector, zero-extended to 32 bits. It does not receive the scaled F-F12 result.

### Final selector source and termination boundary

F-G01 supplies a value to F-G02. F-G03 through F-G05 follow without explicit arguments. F-G06 supplies a 32-bit result.

The function next reads a selector byte at eight times the earlier F-F12 unsigned result plus one, using unsigned 32-bit reduction. At selector + 0x11 it writes bits 20 through 27 of that F-F12 result.

This final read is not inherently a read at T + 1. Its address comes from a different value, although aliasing is possible.

F-G07 receives the F-G06 result shifted left by two. F-G08 through F-G11 follow without explicit arguments. The represented body then reaches the invalid-instruction boundary.

No subsequent cleanup, completion indication, error result, or normal return is established.

## Side effects

### Access order and widths

The function directly performs seven selector-byte reads:

- One from the hidden incoming address plus one.
- Two from Q + 1.
- Three from T + 1.
- One from the final F-F12-derived address.

It performs selector-dependent byte writes, the direct byte clear derived from P, and 32-bit writes at selector-derived, dependency-derived, and absolute addresses.

The four clears at address 0x24 are separate observable writes. They must not be merged merely because they store the same value. Intervening dependencies may observe or modify that address, and calls after the last direct clear may change it again. A final value of zero is not guaranteed.

Repeated selector reads are likewise distinct events. Dependencies or earlier writes may change the selected byte between reads.

### Aliasing and layout

No object layout or disjointness guarantee is established. Input-derived addresses, selector storage, byte destinations, word destinations, and address 0x24 may overlap.

An earlier write may therefore alter a later selector, and a 32-bit store may overlap a byte destination. The retained selector used for the word clear remains the previously read numeric byte even if its original storage changes.

Word byte order and unaligned-access support are **documented_only**. In particular, selector + 0x44 need not be four-byte aligned. The represented 32-bit stores must not be silently replaced with four independently observable byte stores.

### Ownership and failure

No explicit allocation, release, ownership transfer, or lifetime protocol is established in the visible body. Unknown dependencies may perform such operations; their absence cannot be assumed.

There are no visible validation branches, status-dependent recovery paths, or rollback operations. A nonreturning dependency or failed access prevents any guarantee about later events. Earlier effects are not shown to be undone.

## Dependencies

The dependency occurrences are F-A01–F-A12, F-B01–F-B12, F-C01–F-C16, F-D01–F-D22, F-E01–F-E22, F-F01–F-F13, and F-G01–F-G11.

These are opaque occurrence labels, not verified symbol identities. Their targets, complete signatures, and equivalence to one another are unavailable.

For each dependency:

- Only represented explicit arguments and consumed results are specified.
- Additional register, stack, memory, or global inputs remain possible.
- No preservation of globals, selector storage, ownership, or hidden machine state is assumed.
- Purity, termination, non-faulting behavior, and allocation behavior are not established.
- Later events remain conditional on a compatible return.

F-C14 is distinguished by its consumed 64-bit result. F-C16 is distinguished by its use of auxiliary register state following F-C15.

## Acceptance examples

These examples are conditional static-contract cases, not executed tests. They assume the specified calls return, necessary arithmetic is meaningful under the actual convention, and reached accesses succeed.

1. **Unsigned selector.**  
   A selector byte of 0xFF produces byte destination 0x110. It does not produce a sign-extended destination or trigger a pointer-sized load.

2. **Retained selector.**  
   If the second Q-based read yields seven, the immediate byte clear targets 0x18. The later selector-derived word clear targets 0x4B, even if intervening calls change the byte at Q + 1.

3. **Defined unsigned multiplication wrap.**  
   For P = 0xFFC00001 and P = 1, M equals one in both cases. The represented unsigned multiplications also produce identical Q = 0x00000400 and T = 0x04000000. Thus the numeric Q- and T-based selector-read addresses are respectively 0x00000401 and 0x04000001 in both cases. Masked values are 128 and 16 as applicable. The direct P + 0x0F calculations differ: 0xFFC00010 versus 0x00000010. Identical selector-read addresses do not guarantee identical read contents, because prior effects and aliasing may differ.

4. **Direct unsigned addition wrap.**  
   For P = 0xFFFFFFFF, the direct P + 0x0F calculation yields 0x0000000E modulo 2³². Whether that numeric address permits the byte clear is a separate environmental requirement.

5. **Earlier stored value retained.**  
   If F-D22 returns three and F-E05 returns five, the first corresponding store writes 12 at address 0x44. The later store writes the same 12 at address 0x28, rather than replacing it with 20.

6. **Split result.**  
   If F-C14’s low half is three and high half is five, F-C15 receives 12. After it returns, the associated word clear targets 0x4E. This example does not imply that an unaligned write succeeds.

7. **Byte truncation and extraction.**  
   If F-E22 returns 0x12345678, its first associated byte write stores 0xE0, and its later high-field write stores 0x23. Those writes may have different destinations because the selector is reread.

8. **Independent final selector address.**  
   If F-F12 returns two, the final selector read is at 0x11. Its subsequent byte write stores zero, because bits 20 through 27 of two are zero. No equality with T + 1 is implied.

9. **No normal-completion requirement.**  
   Even when all calls and accesses in the prefix succeed, the representation does not establish a normal return after the final dependency.

## Uncertainties

The following remain **documented_only**:

- Actual function boundaries and validity of the represented instruction stream.
- The complete calling convention and legitimacy of hidden entry state.
- Dependency identities, contracts, and effects.
- Potentially overflowing signed arithmetic and signed left shifts.
- Out-of-range unsigned-to-signed conversions and effective-address interpretation.
- Memory mapping, word byte order, alignment requirements, and access-fault behavior.
- Auxiliary register provenance following F-C15.
- Runtime consequences of the invalid-instruction boundary.

These uncertainties do not erase established unsigned arithmetic: Q and T reduce modulo 2³² before assignment, as do the represented unsigned direct P addition and final unsigned selector calculation.

Repeated clearing and bit extraction suggest state manipulation or encoding, but that is inferred intent, not an acceptance requirement.

## Evidence

**Established static evidence:** four explicit arguments; additional incoming-state uses; ordered indirect calls; unsigned byte selectors; byte and 32-bit writes; a consumed 64-bit result; explicit unsigned scaling of P; and an invalid-instruction termination boundary.

**Evidence limitations:** the supplied representation warns of bad instruction data and overlapping global representations. Those warnings limit confidence in recovered boundaries, types, and apparent straight-line execution. The location at 0x24 is specified by address and access width, not by an assumed global-object identity.

**Specification interpretation:** phase boundaries, arithmetic notation, and dependency occurrence IDs organize the observed behavior without establishing application intent or dependency identity.

**Runtime evidence:** none. No runtime tests have been performed.
