<!-- Trusted provenance: F-ab37cb4f6ff8; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-ab37cb4f6ff8

## Purpose

F-ab37cb4f6ff8 performs an ordered sequence of indirect calls, 32-bit memory transfers, arithmetic transformations, and two byte-sized writes. Several call results determine subsequent memory addresses. One 32-bit value is loaded, shifted right by 12 bits, and stored elsewhere. A later call result supplies two separate byte writes through an address selected by a byte-sized lookup.

The supplied static representation does not establish a higher-level purpose. In particular, it does not establish that the addressed locations are objects, buffers, hardware registers, or valid mapped memory.

The representation explicitly reports invalid instruction data and terminates at an invalid-instruction marker. Consequently, this contract describes the visible prefix and its ordering, not a verified complete routine or a guaranteed normal-returning operation.

## Interface

### Entry state

The displayed interface has no declared parameters and no declared return value. Nevertheless, the visible behavior depends on incoming machine state:

| Entry component | Visible use |
|---|---|
| Register R4 | Base for the first 32-bit write, at byte offset 20. |
| Register R7 | Supplies the complete 32-bit payload for that write. |
| Register R3 | Supplies a value shifted left by 26 bits for an early dependency invocation. |
| Link register | Determines the first indirect call target by addition of 4094 bytes. |
| Address space and callable targets | Supply all subsequent call behavior and memory accessibility. |

These are implicit machine-state inputs, not an established public calling convention. Their preservation requirements are unknown.

### Widths and address expressions

The representation uses 32-bit integer values and 8-bit byte accesses:

- Each word store writes 32 bits.
- The single explicit word load reads 32 bits and is subsequently treated as unsigned.
- Each byte lookup reads an unsigned 8-bit value and widens it before address calculation.
- Each byte store retains only the low 8 bits of its computed payload.
- Address offsets are byte offsets.

Multiplication by 4, multiplication by 16, and several left shifts appear in signed-typed expressions. The source representation alone does not establish overflow semantics for those expressions. Numerical examples below avoid signed overflow unless explicitly stated otherwise.

For describing the visible transfers, this contract gives successive result-producing call sites the labels **P1** through **P10**. These are specification labels, not source identifiers:

| Result | Producing dependency site |
|---|---|
| P1 | F-000000000005 |
| P2 | F-00000000000a |
| P3 | F-00000000000f |
| P4 | F-000000000014 |
| P5 | F-000000000019 |
| P6 | F-00000000001e |
| P7 | F-000000000023 |
| P8 | F-000000000028 |
| P9 | F-00000000002d |
| P10 | F-000000000039 |

Two additional results are relevant: a 32-bit forwarding payload from F-00000000002f and a scaling input from F-000000000034.

### Completion and failure interface

No successful return value is established. After the final visible dependency invocation, the representation ends at an invalid-instruction marker.

No explicit error code, validation failure, exception handler, rollback, or cleanup path is visible. A dependency that does not return, or an inaccessible memory access, can prevent the remainder of the prefix from executing.

## Behavior

The order below is the visible caller-side order, conditional on each preceding operation completing. Dependency internals may introduce additional observations and effects.

### Initial publication and first scaled transfer

Before any dependency invocation, the routine stores the incoming R7 value as a 32-bit word at the address formed from incoming R4 plus 20.

It then invokes F-000000000001 through F-000000000005 in order. The first target depends on the incoming link register; the fifth invocation supplies P1.

The routine next stores:

- Address: 16 × P1 + 20.
- Width: 32 bits.
- Payload: 4 × P1.

Only after that store does it invoke F-000000000006 with three visible arguments, in order:

1. 4 × P1.
2. The same 4 × P1.
3. Incoming R3 shifted left by 26 bits.

The two equal arguments are separate argument positions; they do not represent two memory loads.

### First destination selected by a later result

F-000000000007 through F-00000000000a execute in order, with the last producing P2.

A 32-bit store then writes the previously computed 4 × P1 at address 4 × P2 + 20. F-00000000000b follows with the single visible argument 4 × P2.

Thus, the source payload and destination selector originate from different calls. The visible caller does not reload the payload from the earlier destination.

### Second scaled transfer

F-00000000000c through F-00000000000f execute in order and produce P3 at the last site.

The routine stores 4 × P3 as a 32-bit word at address 16 × P3 + 20, then invokes F-000000000010 with 4 × P3.

F-000000000011 through F-000000000014 then execute, with the last producing P4. The routine stores the retained 4 × P3 at address 4 × P4 + 20, followed by F-000000000015 with 4 × P4.

These transfers resemble the preceding pair structurally, but their dependency sites and returned values are distinct.

### Word read and shifted publication

F-000000000016 through F-000000000019 execute in order, with the last producing P5.

The next two caller-side memory operations are ordered as follows:

1. A 32-bit write of 4 × P5 at address 16 × P5 + 20.
2. A 32-bit unsigned read from address 4 × P5 + 20.

The loaded word is retained for a later transfer. These two addresses must not be assumed distinct.

F-00000000001a is then invoked with 4 × P5. F-00000000001b through F-00000000001e follow, with the final site producing P6.

The routine stores the retained loaded word, logically shifted right by 12 bits, as a 32-bit word at address 4 × P6 + 20. The shift fills vacated high bits with zero.

F-00000000001f then receives 4 × P6.

### Additional scaled transfer and bit-field arguments

F-000000000020 through F-000000000023 execute in order, producing P7 at the last site.

The routine stores 4 × P7 as a 32-bit word at address 16 × P7 + 20. It then invokes F-000000000024 with these three visible arguments:

1. The low six bits of 4 × P6, shifted left by 14 bits.
2. 4 × P7.
3. P6 shifted left by 30 bits.

The first argument uses the scaled P6 value, not P6 itself. Its maximum value is 0xFC000 before accounting for any constraints imposed by the preceding multiplication. Under ordinary 32-bit scaling without overflow, the low two bits of 4 × P6 are zero, further restricting possible values.

F-000000000025 through F-000000000028 follow, producing P8 at the last site. The routine stores the retained 4 × P7 at address 4 × P8 + 20, then invokes F-000000000029 with 4 × P8.

### First byte lookup and write

F-00000000002a through F-00000000002d execute in order. The last supplies P9, treated as an unsigned 32-bit value.

The routine obtains a byte from the address formed by:

- Multiplying the earlier P6 by 268435456.
- Adding 1.

That byte is widened without sign extension. Its value, plus 17, becomes the destination address for a byte store. The payload is the low byte of P9 shifted left by 2 bits.

Consequently:

- The destination lies numerically between 17 and 272, inclusive.
- The payload has its low two bits clear.
- This is a byte-derived numerical address, not an established pointer-table lookup.

F-00000000002e follows with three visible arguments:

1. P9 shifted left by 2 bits.
2. The same shifted value.
3. P9 shifted left by 4 bits.

This site uses a target displacement of 4076 bytes from its fixed anchor, unlike the 4094-byte displacement shown for the other anchor-relative calls.

### Forwarded result and second byte write

F-00000000002f produces a 32-bit payload that is passed directly as the visible argument to F-000000000030.

F-000000000031 through F-000000000034 then execute in order. The last supplies the scaling input used by the next argument-bearing call.

After these calls, the routine reads the byte at the same lookup address derived from P6 again. This is a fresh memory read; the earlier lookup value is not reused.

The newly read byte, widened and increased by 17, selects the second byte-store destination. The payload is the low byte of:

- P9 masked with 0x3FFFFFFF.
- Then logically shifted right by 20 bits.

After byte truncation, this payload corresponds to bits 20 through 27 of P9. It is not derived from the intervening forwarding payload or the result of F-000000000034.

F-000000000035 then receives the F-000000000034 result shifted left by 2 bits.

### Final visible calls

F-000000000036 through F-000000000039 execute in order, with the last producing P10.

F-00000000003a receives two visible arguments:

1. P10 shifted left by 2 bits.
2. The widened byte obtained by the second lookup, without the added destination offset.

F-00000000003b is then invoked with no explicit arguments shown. The visible prefix ends at the invalid-instruction marker.

## Side effects

The visible prefix contains:

- Nine explicit 32-bit stores.
- One explicit 32-bit load.
- Two explicit unsigned byte loads.
- Two explicit byte stores.
- Fifty-nine indirect dependency invocations.

These counts exclude dependency-internal operations and any machine effects not represented in the supplied material.

There are no explicit allocation, deallocation, ownership-transfer, or lifetime-management operations. Passing a numerical address-like value to a dependency does not establish that the dependency receives ownership.

No non-aliasing guarantee is visible. Earlier stores can affect later loads when their addresses coincide. Dependencies may also modify those locations, globals, or other state. In particular, the two byte lookups must remain distinct observations because calls occur between them.

No atomicity, synchronization, memory barrier, volatile-access classification, or thread-safety guarantee is established. Width alone does not prove hardware atomicity.

## Dependencies

The opaque IDs F-000000000001 through F-00000000003b identify the 59 visible invocation sites in execution order. They are local contract identifiers, not verified identities from an external dependency registry. Distinct site IDs do not prove distinct runtime callees.

The first site uses an entry-state-derived target. The remaining sites use fixed anchors with displayed displacements. Callee bodies and calling conventions are unavailable.

Sites with visible argument lists are:

| Site | Visible argument role |
|---|---|
| F-000000000006 | First scaled payload twice; shifted entry R3. |
| F-00000000000b | Scaled P2. |
| F-000000000010 | Scaled P3. |
| F-000000000015 | Scaled P4. |
| F-00000000001a | Scaled P5. |
| F-00000000001f | Scaled P6. |
| F-000000000024 | Masked/scaled P6 field; scaled P7; shifted P6. |
| F-000000000029 | Scaled P8. |
| F-00000000002e | Two equal shifted P9 arguments; another shifted P9 argument. |
| F-000000000030 | Result forwarded from the immediately preceding site. |
| F-000000000035 | Shifted result from F-000000000034. |
| F-00000000003a | Shifted P10; second lookup byte. |

“No explicit arguments shown” does not establish that a callee consumes no incoming registers or other implicit state.

## Acceptance examples

These are conditional contract examples, not executed tests. Each assumes that all preceding calls return, required addresses are accessible, arithmetic is representable where signed behavior is involved, and dependencies do not invalidate the stated setup.

### Different scaling for store address and payload

If P1 is 3, the initial P1-derived word store targets address 68 and writes 12.

If P2 is subsequently 5, the following destination-selected store targets address 40 and also writes 12. F-00000000000b receives 20.

### Store-before-load aliasing

If P5 is zero, both its word-store address and subsequent word-load address are 20. The store writes zero before the read.

Assuming ordinary readable/writable memory at that address, the retained word is therefore zero, and its later right-shifted publication is zero. Treating the load as occurring before the store would violate the visible order.

### Unsigned right shift

If the retained word is 0xFEDCBA98, the subsequent 12-bit logical right shift produces 0x000FEDCB. A sign-extending result would not satisfy this contract.

### Independent byte lookups

Suppose P6 is zero and P9 is 0x12ABCDEF. If the first lookup at address 1 yields 5:

- The first byte-store destination is 22.
- Its payload is 0xBC.

If intervening dependencies change the lookup byte to 9:

- The second destination is 26.
- Its payload is 0x2A.
- F-00000000003a receives 9 in its second visible argument position, not 5 and not 26.

### Incomplete dependency execution

If F-000000000006 does not return, the entry-state word store and the P1-derived word store precede that invocation. No later caller-side operation is established as occurring. No rollback is visible.

## Uncertainties

The following are **documented_only**:

- Complete instruction validity, control-flow recovery, and the meaning of the terminal invalid-instruction marker.
- The actual public ABI and register preservation rules.
- Exact runtime callee identities, signatures, effects, and failure behavior.
- Signed-overflow behavior, pointer width, address wrapping, and left-shift semantics beyond representable cases.
- Endianness and alignment requirements.
- Whether any addressed memory is mapped, ordinary RAM, device memory, or protected.
- Ownership, allocation provenance, and lifetime of referenced storage.
- Normal return, downstream execution, and any externally defined error protocol.

A possible intent is staged publication of scaled values and extracted fields. That is an inference only. The evidence does not justify naming a protocol, data structure, subsystem, or resource lifecycle.

## Evidence

Established static evidence consists solely of the supplied representation: its implicit entry-state dependencies, ordered indirect invocations, explicit access widths, arithmetic expressions, retained call results, and invalid-instruction warning.

The contract distinguishes caller-side operations from unknown callee behavior and does not assume preservation of globals or ownership conventions.

No runtime tests have been performed. The acceptance examples are arithmetic and ordering consequences of the visible prefix under their stated preconditions, not observations of executable behavior.
