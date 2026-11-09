<!-- Trusted provenance: F-4cafe00cfb76; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-4cafe00cfb76

## Purpose

F-4cafe00cfb76 describes an ordered prefix of memory accesses and indirect calls. It uses one integer input both as a stored value and as an address component. Selected dependency results determine later addresses, stored values, and call arguments.

The supplied representation does not establish a higher-level purpose. Array indexing, object manipulation, device access, and protocol processing are possible interpretations, not established behavior.

The visible prefix ends at a bad-instruction boundary, without an ordinary return. A control-flow reliability warning also applies to the supplied representation. This contract therefore distinguishes the displayed operations from any claim that they constitute a verified executable path.

## Interface

### Input and ambient context

The recovered interface presents one signed integer input, designated **P**. Its visible uses are:

- The value of an initial four-byte write.
- An address component scaled by four for that write.
- An unscaled address component for the following four-byte read.
- The one explicit argument to F-D01.

No input validation, range check, alignment check, or access-permission check appears before these uses.

An ambient link-related execution value contributes to the first indirect call target. It is not an explicit argument in the recovered interface. Its origin and validity are unknown.

### Width and arithmetic conventions

The representation distinguishes four-byte integer accesses, unsigned one-byte reads, and one-byte writes.

This contract uses the following notation to describe the recovered operations:

- A **word** is a 32-bit bit pattern.
- A **word result** retains the low 32 bits of an arithmetic or left-shift result.
- A logical right shift inserts zero bits.
- A byte read produces an unsigned value from 0 through 255.
- A byte write stores the low eight bits of its supplied value.

These conventions describe the bit-level interpretation used below. They do not establish portable signed-arithmetic behavior. Some recovered expressions use signed integer arithmetic, including multiplication and left shifts. Overflow, negative operands, conversion to addresses, and address wrapping require the actual execution model to be known.

For address calculations near a width boundary, the exact effective address is **documented_only**. Numerical examples below either avoid that boundary or explicitly depend on the 32-bit interpretation.

No byte order, alignment requirement, or memory-region layout is independently established. Four-byte accesses describe their width, not a guarantee of atomicity.

### Output, completion, and errors

No ordinary return value is established. After F-D34, the visible representation terminates at a bad-instruction boundary.

There is no visible local error result, exception handler, rollback, or recovery path. If an access cannot complete, or a dependency does not return normally, subsequent operations are not established to occur. The function does not visibly undo earlier completed writes.

Reaching the end of the displayed prefix requires valid memory accesses, callable indirect targets, and normally returning dependencies. These are environmental requirements, not checks implemented by the visible prefix.

## Behavior

### Values and call-site identities

F-D01 through F-D34 identify indirect call sites in displayed execution order. They are opaque, document-local identifiers. Separate IDs do not prove that the targets are different implementations.

The following names identify captured values, not memory locations:

| Value | Origin | Principal later uses |
|---|---|---|
| S | Initial four-byte read | Logical right shift for a later store |
| A | F-D05 result | Address calculations and call arguments |
| B | F-D10 result | Scaled stored value and first destination for that value |
| C | F-D15 result | Second destination for the scaled B value |
| D | F-D20 result | Both byte-write values and F-D21 arguments |
| E | F-D22 result | Argument to F-D23 |
| F | F-D27 result | Scaled argument to F-D28 |
| Q | Second unsigned byte read | Second byte-write destination and argument to F-D33 |
| G | F-D32 result | Scaled argument to F-D33 |

E is retained as a four-byte result without an established signed interpretation. Other named call results are interpreted through their visible word-sized uses.

### Initial access pair and F-D01 through F-D05

The first direct memory operation writes four bytes containing P to the address formed by four times P plus 0x14.

The next direct operation reads four bytes from the address P plus 0x14, interpreting the result as an unsigned word and retaining it as S.

The write precedes the read. Their locations are not established to be disjoint. Complete or partial overlap can make S depend on the preceding write. The exact effect of partial overlap depends on byte order and the memory environment.

The function then invokes F-D01 with P as its explicit argument, followed by F-D02, F-D03, F-D04, and F-D05. The result of F-D05 is captured as A.

No explicit arguments are shown for F-D02 through F-D05. That does not establish that their real interfaces lack hidden inputs, register inputs, or access to shared state.

### Saved-word store and F-D06 through F-D10

The function forms the word result of four times A.

It writes S logically shifted right by 12 as a four-byte value to the address formed by four times A plus 0x14.

This store uses the previously captured S. There is no intervening refresh of S from its original address. Dependency changes to that address therefore do not, by themselves, replace the value used by this store.

F-D06 receives the word result of four times A. F-D07, F-D08, F-D09, and F-D10 follow in order. The result of F-D10 is captured as B.

The function forms the word result of four times B and writes that four-byte value to the address formed by sixteen times B plus 0x14.

The value scaling and destination scaling differ: the stored value uses a factor of four; the destination uses a factor of sixteen.

### Three-argument call and repeated B-derived store

F-D11 receives three explicit arguments, in order:

1. The low six bits of the word result of four times A, shifted left by 14.
2. The word result of four times B.
3. The word result of A shifted left by 30.

Under the 32-bit interpretation, the first argument depends on bits 0 through 3 of A. Multiplication by four clears the two lowest bits before the six-bit mask is applied. The argument is therefore one of sixteen multiples of 0x10000, from zero through 0xF0000.

The third argument depends only on the two lowest bits of A.

F-D12, F-D13, F-D14, and F-D15 follow in order. The F-D15 result is captured as C.

The function then writes the previously formed word result of four times B as four bytes to the address formed by four times C plus 0x14. It does not obtain a new B value.

The two B-derived stores need not share a destination. The earlier one uses sixteen times B; the later one uses four times C.

### First selector read and byte write

F-D16 receives the word result of four times C. F-D17, F-D18, F-D19, and F-D20 follow in order. The result of F-D20 is captured as D.

The function reads one unsigned byte from the address formed by A multiplied by 0x10000000, plus one. Under 32-bit arithmetic, this address expression depends only on the lowest four bits of A.

The byte is used as a numeric destination selector. It is not a full-width pointer loaded from memory.

The function adds 0x11 to that selector and writes one byte at the resulting address. The stored byte is the low eight bits of the word result of D shifted left by two.

Consequences of these widths are:

- The destination ranges from 0x11 through 0x110, inclusive.
- The stored byte is a multiple of four.
- Only bits 0 through 5 of D influence that byte.

No selector validation is visible.

### F-D21 through F-D27 and second byte write

F-D21 receives three explicit arguments:

1. The word result of D shifted left by two.
2. The same word result again.
3. The word result of D shifted left by four.

F-D22 follows, with its four-byte result captured as E. F-D23 receives E as its explicit argument. F-D24, F-D25, F-D26, and F-D27 then execute in order. The result of F-D27 is captured as F.

After F-D27, the function performs a second unsigned byte read using the same address construction as the first: A multiplied by 0x10000000, plus one. This newly observed byte is captured as Q.

This is a separate memory access. The first selector is not reused. Intervening dependencies, earlier writes, or external activity may affect the second observed value.

The function writes one byte to Q plus 0x11. Its value is formed by retaining the low 30 bits of D, logically shifting right by 20, and retaining the low eight bits for the write.

The observable stored byte is therefore bits 20 through 27 of D. Bits 28 and 29 contribute to the wider intermediate result but are discarded by the byte write.

The two byte writes need not select the same destination.

### Final calls and boundary

The final call sequence is:

1. F-D28, receiving the word result of F shifted left by two.
2. F-D29.
3. F-D30.
4. F-D31.
5. F-D32, whose result is captured as G.
6. F-D33, receiving the word result of G shifted left by two, followed by Q.
7. F-D34.

F-D33 receives the saved second selector Q. No third selector read is shown before that call.

The representation then reaches the bad-instruction boundary. No ordinary return, successful completion status, cleanup, or behavior beyond that boundary is established.

## Side effects

### Direct memory-access order

Subject to all preceding operations completing, the direct accesses occur in this order:

| Order | Access | Address construction | Value |
|---|---|---|---|
| 1 | Four-byte write | Four times P, plus 0x14 | P |
| 2 | Four-byte read | P plus 0x14 | Captured as S |
| 3 | Four-byte write | Four times A, plus 0x14 | S logically shifted right by 12 |
| 4 | Four-byte write | Sixteen times B, plus 0x14 | Word result of four times B |
| 5 | Four-byte write | Four times C, plus 0x14 | Same scaled B value |
| 6 | Unsigned byte read | A times 0x10000000, plus one | First selector |
| 7 | Byte write | First selector plus 0x11 | Low byte of D shifted left by two |
| 8 | Unsigned byte read | Same construction as access 6 | Captured as Q |
| 9 | Byte write | Q plus 0x11 | Bits 20 through 27 of D |

Dependencies are interleaved as specified in Behavior. This table does not exclude dependency accesses or imply that memory remains unchanged between direct accesses.

### Ownership and state

The direct operations show no allocation, deallocation, ownership transfer, locking, or resource release.

No ownership rule can be inferred for the computed addresses. Dependency implementations may access globals, mutate shared state, allocate or release resources, or fail to return. They are not assumed to preserve globals or memory.

No disjointness guarantee applies among direct-access locations, dependency-managed storage, or external state. There is no visible transaction boundary or restoration of earlier memory contents.

## Dependencies

There are 34 displayed indirect call sites.

| Opaque IDs | Explicit arguments and retained results |
|---|---|
| F-D01 | P |
| F-D02–F-D04 | No explicit arguments shown |
| F-D05 | Result captured as A |
| F-D06 | Word result of four times A |
| F-D07–F-D09 | No explicit arguments shown |
| F-D10 | Result captured as B |
| F-D11 | Masked-and-shifted scaled A; scaled B; A shifted left by 30 |
| F-D12–F-D14 | No explicit arguments shown |
| F-D15 | Result captured as C |
| F-D16 | Word result of four times C |
| F-D17–F-D19 | No explicit arguments shown |
| F-D20 | Result captured as D |
| F-D21 | D shifted left by two; same again; D shifted left by four |
| F-D22 | Four-byte result captured as E |
| F-D23 | E |
| F-D24–F-D26 | No explicit arguments shown |
| F-D27 | Result captured as F |
| F-D28 | F shifted left by two |
| F-D29–F-D31 | No explicit arguments shown |
| F-D32 | Result captured as G |
| F-D33 | G shifted left by two; Q |
| F-D34 | No explicit arguments shown |

Left-shifted arguments in this table use the word-result convention.

F-D01’s target is formed from the ambient link-related value plus 0xFFE. Subsequent targets are formed from embedded target material plus an offset. The displayed offset is 0xFFE except at F-D21, which uses 0xFEC.

This target-construction pattern does not establish target validity, semantic identity, complete signatures, ownership behavior, or effects. Results not captured by the visible caller do not imply effect-free calls.

## Acceptance examples

These are conditional deductions from the supplied static representation, not executed tests. They require valid accesses, normal dependency returns, and the stated arithmetic interpretation.

1. **Initial full overlap.**  
   With P equal to zero, both initial address expressions select 0x14. In ordinary memory without intervening external mutation, the four-byte zero write precedes the read, so S is zero. Later changes to that location do not refresh S.

2. **Logical right shift.**  
   If the initial read captures S as 0xABCDEF12, the later S-derived store writes 0x000ABCDE. It uses the captured value even if a dependency subsequently changes the original location.

3. **F-D11 argument calculation.**  
   With A equal to five and B equal to three, four times A is 0x14. Masking to six bits leaves 0x14; shifting left by 14 gives 0x50000. F-D11 therefore receives **(0x50000, 12, 0x40000000)**.

4. **Different scaling for destinations.**  
   With B equal to three and C equal to seven, the stored B-derived word is 12. Its first destination is 0x44; its later destination is 0x30.

5. **Independent selector observations.**  
   With A equal to two, the selector-read address is 0x20000001. If the first read observes 0x20 and the second observes 0x40, the byte-write destinations are 0x31 and 0x51. F-D33 later receives 0x40 as its second explicit argument.

6. **Byte truncation.**  
   With D equal to 0x3FF00003, the first byte write stores 0x0C. The second stores 0xFF. The wider intermediate value for the second write is 0x3FF, but only its low byte is written.

7. **Selector limits.**  
   A selector of zero selects destination 0x11. A selector of 255 selects destination 0x110; the addition is not an eight-bit wrap to 0x10.

8. **Early access failure.**  
   If the initial four-byte write cannot complete, the representation establishes no subsequent read or call on that execution. No local returned error code or rollback is shown.

## Uncertainties

The following remain **documented_only**:

- Whether the displayed operations form a valid executable path throughout.
- Actual architecture, calling convention, address width, byte order, alignment, and arithmetic behavior outside representable signed ranges.
- The meaning and valid domain of P.
- The validity and intended layout of each computed memory region.
- Dependency identities, complete interfaces, hidden inputs, effects, ownership rules, and failure modes.
- Memory aliasing, concurrent mutation, memory-mapped I/O behavior, and access atomicity.
- The runtime consequence of the bad-instruction boundary.
- Any behavior after that boundary.

Scaled addressing and bit extraction suggest an indexing or field-manipulation pattern. That is inferred intent only; it does not establish an array, object layout, register bank, or protocol.

## Evidence

Established static evidence is limited to the supplied recovered representation: ordered four-byte and one-byte accesses, arithmetic and bitwise expressions, 34 indirect call sites, selected retained results, and a terminal bad-instruction indication.

The representation includes a control-flow reliability warning. The contract preserves the displayed ordering without asserting independently verified reachability of every operation.

The F-D11 example is derived from the displayed scaling, six-bit mask, and 14-bit shift. For A equal to five, that calculation produces 0x50000.

No runtime tests have been performed. No dependency implementations, external callers, memory map, or execution trace were supplied. All acceptance examples are conditional static consequences, not observed executions.
