<!-- Trusted provenance: F-3b46f8ee42fa; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-3b46f8ee42fa

## Purpose

F-3b46f8ee42fa is an externally state-dependent routine that makes a long, ordered sequence of indirect calls and interleaves them with byte and 32-bit memory accesses. Its visible effects include scaled-address writes, masked and shifted value propagation, byte-indexed writes into low memory, and writes to the absolute addresses 0x0D and 0x11.

No application-level purpose is established. In particular, the available evidence does not establish that the routine initializes an object, manages a collection, transfers ownership, or implements a particular protocol.

This contract describes the visible trace conditionally: execution must reach each described operation, its indirect targets must be callable, and its memory accesses must be valid in the execution environment. The supplied representation reports invalid instruction data and ends at an invalid-instruction termination marker. It therefore does not establish an ordinary successful return or complete behavior beyond that boundary.

## Interface

**Explicit input.** One value represented as a signed 32-bit integer. Call this the **input address**. It is:

- Passed unchanged to the first indirect dependency.
- Later used as the address base for a one-byte write at offset 1.

Treating it as an address is required to describe that access; it does not establish a valid object, allocation extent, or ownership relationship.

**Additional entry state.** The routine visibly depends on:

- An incoming link-state value used to calculate its first indirect-call target.
- An externally supplied unsigned 32-bit value, called the **entry field** below.
- Another externally supplied unsigned 32-bit value, called the **entry auxiliary**.
- Secondary machine-result values from particular dependency calls.
- Memory and all state observable by the indirect dependencies.

These are not established as additional formal arguments. Their calling-convention significance is unknown.

**Output.** The represented interface has no explicit return value. No normal return is shown. Outputs consist of memory effects and dependency interactions before termination or an earlier fault.

**Width conventions.**

- Bytes are 8 bits; words are 32 bits.
- Byte stores retain only the low 8 bits.
- Unsigned right shifts insert zeroes.
- Masks and unsigned left shifts operate on 32-bit values.
- Two dependency results are represented as pairs of 32-bit words; another such pair occurs earlier.
- Address calculations involving signed multiplication or signed left shift require qualification. The evidence suggests machine-word arithmetic, but it does not establish portable signed-overflow semantics. Examples below avoid those ambiguities unless explicitly conditional.
- Endianness and alignment requirements are not established.

**Preconditions for a complete visible trace.** All reached indirect targets and memory accesses must be valid. This includes low-address memory, not merely storage associated with the input address. No visible validation establishes these preconditions.

## Behavior

### Dependency and value notation

Opaque IDs F-D001 through F-D009 below identify selected call occurrences, not resolved function identities. They are assigned only to make this contract readable. They do not extend the supplied dependency map or imply distinct callees.

The following descriptive values denote captured results, not source identifiers:

| Value | Origin |
|---|---|
| First word | Result of F-D002 |
| First index | First later result used to form a scaled memory address |
| Early secondary | Secondary machine result used after the initial address-based call sequence |
| Entry slice | Entry field logically shifted right by 22 |
| Record seed | Later result first used to write at sixteen times itself plus 0x14 |
| Record mask | Record seed masked with 0x3FFFFFFF |
| Record quarter | Record mask logically shifted right by 2 |
| Table index | Later result used first at four times itself plus 0x14 |
| Payload seed | Result captured after the table-index notification sequence |
| Payload base | Payload seed masked with 0x3FFFFFFF |
| Eightfold base | Eight times table index |
| Sixteenfold base | Sixteen times table index |

Captured results remain the values used by subsequent direct operations. Memory is different: dependencies may change it between accesses. Nothing here assumes that a dependency preserves globals, preserves reachable storage, or allocates or releases memory.

### Entry and first word propagation

The first indirect call, F-D001, receives the input address. Its target depends on incoming link state.

F-D002 is then called without explicit arguments and supplies the first word. The routine derives the entry slice and invokes F-D003 with:

1. The first word.
2. The entry auxiliary logically shifted right by 2.

Three further calls without explicit arguments precede the call supplying the first index.

The routine then performs, in order:

1. A 32-bit read at four times the first index plus 0x74.
2. A byte write at four times the first index plus 1, containing the low byte of that read.
3. An indirect call receiving four times the first index.

After four intervening calls without explicit arguments, it writes the same captured 32-bit word at four times the early secondary plus 0x14. A dependency then receives the early secondary.

A subsequent result-producing call supplies an argument to F-D004. F-D004 receives that result and two identical arguments, each equal to the low nine bits of the entry slice shifted left by 21.

Since the entry slice contains at most ten bits, this argument deliberately discards its highest possible bit.

### Entry-derived and record-derived writes

After three calls without explicit arguments and a result-producing call, the routine:

- Writes the entry slice shifted left by 23 to an address four times that result plus 0x14.
- Calls a dependency with four times that result.

After three further calls without explicit arguments, a call supplies the record seed.

The next direct store writes four times the record seed as a 32-bit value at sixteen times the record seed plus 0x14. This is not an object-relative store unless the environment independently gives that interpretation to the resulting numeric address.

The record mask and record quarter are then derived. A dependency receives the record mask. Three calls without explicit arguments precede the call supplying the table index.

The routine writes the record mask at four times the table index plus 0x14 and calls a dependency with four times the table index.

After three calls without explicit arguments, a call supplies the payload seed. The next operations are ordered:

1. Write the low byte of the eightfold base at input address plus 1.
2. Write the payload seed shifted left by 2 as a 32-bit value at eightfold base plus 0x34.
3. Call a dependency with the payload seed shifted left by 2.

### Payload-base writes and shared-destination replacement

A call without explicit arguments supplies a word, which is passed to the next dependency. The payload base is the low 30 bits of the payload seed.

After three further calls without explicit arguments, a dependency supplies a two-word result. The routine then:

1. Writes the low byte of the eightfold base at payload base plus 1.
2. Writes the high result word at eightfold base plus 0x34.
3. Calls a dependency with the low result word shifted left by 2.

The second write uses the same numeric destination as the earlier payload-seed store. It is a later write, not proof that the earlier value remained intact between stores.

After three calls without explicit arguments, another result is obtained. Before passing twice-identical arguments derived from that result, the routine writes:

- The low byte of the sixteenfold base at  
  ((payload seed masked with 0x3FFFFFFF) shifted right by 20) plus 0x11.
- The low byte of four times the record seed at sixteenfold base plus 0x0D.

The following dependency receives two identical arguments: the newly obtained result shifted left by 2.

### Repeated byte fields and a new scaled record

A result-producing call supplies a word. F-D005 receives that word and the same word logically shifted right by 3.

Three calls without explicit arguments precede a new result. The routine then:

1. Writes the low byte of the sixteenfold base to absolute address 0x11.
2. Repeats the byte write of four times the record seed at sixteenfold base plus 0x0D.
3. Calls a dependency with the new result shifted left by 2.

After three calls without explicit arguments, another result is captured; call it the **wide seed**. Before using it for new addresses, the routine writes:

- Four times the record seed as a 32-bit value at payload base plus 0x14.
- The low byte of twice the payload base at payload base plus 0x0D.

Define:

- **Wide base:** wide seed multiplied by 32.
- **Wide payload:** wide seed shifted left by 7.
- **Small index:** wide seed masked with 0x07FFFFFF, then logically shifted right by 17.

The small index is therefore between 0 and 1023.

Two calls without explicit arguments precede a call supplying the **shift source**. The routine writes the wide payload at wide base plus 0x14, then writes the low byte of the shift source shifted left by 2 at wide base plus 0x0D. A dependency receives that shifted value.

### Byte-indexed low-memory updates

Four calls without explicit arguments precede these direct operations:

1. Write the low byte of twice the record quarter at small index plus 7.
2. Read one unsigned byte at wide base plus 1.
3. Write the low byte of the small index at the address obtained by adding 0x11 to the byte just read.
4. Call F-D006 with three identical arguments, each the shift source shifted left by 21.

The byte-indexed destination lies between 0x11 and 0x110.

A result-producing call supplies the **later word**, which is passed to another dependency. Three calls without explicit arguments precede another result-producing call. The routine then re-reads the byte at wide base plus 1 and writes zero at that byte plus 0x11. The subsequent dependency receives the latest result shifted left by 2.

This second byte read is independent of the first. Intervening dependencies or aliased writes may have changed it.

After four calls without explicit arguments:

- The low byte of the later word logically shifted right by 3 is written at wide base plus 0x0D.
- The byte from the second byte read is written to absolute address 0x0D.
- F-D007 receives twice that byte, 0x98, and 0x260.

Call twice that captured byte the **doubled byte base**. It ranges from 0 to 510.

### Low-memory marker and linked word updates

A call without explicit arguments precedes a dependency receiving a secondary machine result logically shifted right by 3.

After three more calls without explicit arguments, a result-producing call supplies an argument for another result-producing dependency. That argument is shifted left by 2 before the call. Then:

1. The previously derived byte from the later word is written at doubled byte base plus 0x0D.
2. Absolute address 0x0D is written with 0xC0.
3. A dependency receives its predecessor’s result shifted left by 2.

After three calls without explicit arguments, another call supplies a result. The routine:

- Writes that result shifted left by 2 at doubled byte base plus 0x24.
- Reads a 32-bit word at doubled byte base plus 0x74.
- Calls F-D008 with four arguments: the read word; that word shifted left by 1; the latest result shifted left by 5; and the read word masked with 0x7FFFFFFF then logically shifted right by 2.

Call the captured read word the **linked base**.

After four calls without explicit arguments, the routine writes the saved result shifted left by 5 at linked base plus 0x24. It then reads a 32-bit word at linked base plus 0x74 and passes that word to the next dependency.

### Final paired results and repeated clears

Three calls without explicit arguments precede a two-word result. The routine:

- Writes the later word logically shifted right by 3 at four times the high result word plus 0x14.
- Calls F-D009 with three copies of the low result word shifted left by 2, followed by the high result word logically shifted right by 3.

Another two-word result follows. Its low word, high word, low word again, and high word logically shifted right by 3 are passed to the next dependency.

After three calls without explicit arguments, a result-producing call supplies an index. The routine writes the earlier paired result’s low word shifted left by 2 at four times this index plus 0x14, then calls a dependency with four times the index.

After three calls without explicit arguments, a call supplies the **final seed**. Define the **final scaled value** as that seed shifted left by 2.

The remaining visible memory operations, with dependencies between them, are:

1. Read a byte at linked base plus 1; write the low byte of the final scaled value at that byte plus 0x11.
2. After two calls without explicit arguments and a result-producing call, re-read linked base plus 1; write the low byte of the final seed masked with 0x3FFFFFFF and shifted right by 20 at that byte plus 0x11.
3. Pass the intervening result shifted left by 2 to a dependency.
4. After three calls without explicit arguments and a result-producing call, read a byte at final scaled value multiplied by 1024 plus 1; write zero at that byte plus 0x11.
5. Call a dependency with the final scaled value shifted left by 3 twice, the final scaled value, and the latest result shifted left by 2.
6. After three calls without explicit arguments and a result-producing call, repeat the byte read and zero write from step 4.
7. Pass the latest result shifted left by 2 to a dependency, then make four calls without explicit arguments.

The visible representation then ends at invalid instruction data.

## Side effects

All specified direct memory operations are observable parts of the conditional trace. Equal addresses do not make repeated writes or reads interchangeable: dependencies execute between many of them, and accesses may involve mapped devices.

Aliasing is unrestricted by the evidence. Input-relative, scaled, byte-indexed, and absolute-address accesses may overlap. In particular, a write may change a byte used to select a later destination.

No visible allocation, release, reference-count operation, rollback, error check, or ownership transfer is established. Unknown dependencies may perform such actions. Their effects must not be assumed absent.

No explicit status code, recoverable error path, or cleanup path is shown. A dependency fault, invalid address, alignment fault, or invalid instruction may prevent later effects. Earlier effects are not visibly undone.

## Dependencies

The only supplied resolved identity is F-3b46f8ee42fa itself.

| Opaque occurrence ID | Visible contract |
|---|---|
| F-D001 | Receives the explicit input; target depends on entry link state. |
| F-D002 | Supplies the first captured word. |
| F-D003 | Receives that word and shifted entry auxiliary. |
| F-D004 | Receives a result and two identical entry-derived bitfields. |
| F-D005 | Receives a word and its logical right shift by 3. |
| F-D006 | Receives three identical shifted arguments. |
| F-D007 | Receives doubled captured byte, 0x98, and 0x260. |
| F-D008 | Receives the linked-base read and three derived arguments. |
| F-D009 | Receives three identical low-word-derived arguments and a shifted high word. |

Other call occurrences remain unresolved. Calls described as having no explicit arguments may still consume machine state. Ignored results do not imply absent side effects.

## Acceptance examples

These are conditional static examples, not executed tests.

- **Byte truncation:** If the first index is 0x20 and the word read at 0xF4 is 0x123456AB, the next direct write places byte 0xAB at 0x81. The captured word later written through the early secondary remains 0x123456AB.
- **Entry-field masking:** If the entry field is 0xFFFFFFFF, the entry slice is 0x3FF. Each duplicated F-D004 bitfield argument is 0x3FE00000. The later unsigned shift by 23 produces 0xFF800000, not an unbounded-precision value.
- **Shared destination:** If the table index is 4, both writes through eightfold base plus 0x34 target 0x54. The first contains shifted payload seed; the second contains a later paired result’s high word.
- **Re-read matters:** If wide base plus 1 contains 7 at the first byte read and 9 at the second, the small-index byte is written at 0x18 and zero is later written at 0x1A. The captured byte used for the doubled byte base is 9.
- **Absolute marker:** If execution reaches the marker store, address 0x0D receives 0xC0 regardless of the previously stored captured byte. Later dependency effects remain unknown.
- **No null rejection:** A zero input is not visibly rejected before the first dependency. Reaching the later input-relative byte store requires address 1 to be writable; failure behavior is environment-dependent.

## Uncertainties

**Established:** visible access widths, masks, unsigned shifts, argument relationships, direct access order, repeated reads, repeated writes, and the reported invalid-instruction boundary.

**Inferred intent:** recurring offsets may describe record fields, while scaled integers may encode handles or addresses. Neither interpretation is required or established.

**documented_only:**

- Exact target identities, calling convention, and dependency semantics.
- Reliability of the represented control flow around invalid instruction data.
- Machine behavior for signed overflow and signed left shifts.
- Address-space mapping, endianness, alignment, and access atomicity.
- Whether secondary results are intentional outputs or representation artifacts.
- Whether paired results are actual 64-bit returns or multiple machine-register outputs.
- Memory lifetime and ownership across dependencies.
- Terminal behavior at the invalid-instruction boundary.

## Evidence

This contract uses only the supplied static representation. No implementation access, runtime execution, or tests were performed.

The representation supports conditional trace claims, not a verified application-level purpose or successful execution. No snapshot hash or independent provenance claim is included.
