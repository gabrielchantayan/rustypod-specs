<!-- Trusted provenance: F-aaf4359c08ba; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-aaf4359c08ba

## Purpose

F-aaf4359c08ba accepts one explicit 32-bit value and performs an ordered state transformation comprising unresolved dependency calls and interleaved 8-, 16-, and 32-bit memory accesses.

**Established static evidence:** the supplied representation has one visible path, with no explicit conditional branch or loop. Its first observable operation is a word store through an incoming execution-context base. Its final operations clear a byte, compose and store a word at a fixed location, clear a halfword, and invoke address zero.

**Inferred intent:** recurring offsets, scaled destinations, and selector-mediated byte writes suggest structured state propagation. They do not establish an application domain, object type, initialization protocol, or success condition.

This contract describes the visible behavior, conditional on execution reaching each operation. It does not supply contracts for unresolved dependencies or guarantee normal completion.

## Interface

### Explicit input and output

- The sole explicit input is an uninterpreted 32-bit value, called the **input value** below.
- No return value is declared.
- The first dependency receives four visible arguments: the input value, the input value, zero, and the input value.
- No visible validation limits the input value.
- No explicit status result or error-reporting channel is established.

A declared absence of a return value does not establish register preservation.

### Implicit execution inputs

The entry behavior additionally depends on:

- an incoming base used for the initial word-store destination;
- an incoming 32-bit word used as the initial stored payload;
- an incoming execution-context value used to calculate the first call target.

These are implicit inputs, not established additional public parameters. Their origins and ABI requirements are unknown.

Later expressions consume primary dependency results, 64-bit results divided into low and high 32-bit portions, secondary-register values, and byte-sized results. Secondary-register values are not independently established as valid pointers, conventional second return values, or values preserved across arbitrary calls.

### Widths and arithmetic

| Quantity or access | Established width |
|---|---:|
| Explicit input and word payload | 32 bits |
| Byte access | 8 bits |
| Halfword access | 16 bits |
| Word access | 32 bits |
| Split dependency result | 64 bits |
| Each portion of a split result | 32 bits |
| Upper component of the final word composition | 24 bits |

Explicit unsigned-byte reads produce values from zero through 255. Byte stores retain the low eight payload bits. Reading a word and storing its low byte is numeric truncation; it is not necessarily a copy of the byte at the lowest source address. Byte order is not supplied.

Arithmetic deductions below use **32-bit bit-vector interpretation**: addition, multiplication, and left shifts retain the low 32 bits; unsigned right shifts insert zeros. This interpretation does not establish portable semantics for overflowing signed expressions in the supplied representation. Actual address-width and signed-overflow behavior remain documented_only.

### Preconditions for progress

Every reached memory access requires a mapped, permitted address and any required alignment. Every reached call requires an executable target and an ABI-compatible execution context. Continuing past a call requires that dependency to return.

No visible guard enforces these conditions. The function does not visibly validate addresses, selector values, secondary-register state, or the terminal target.

## Behavior

### Observation model

Visible calls and memory operations retain their represented order. Dependencies may mutate memory or globals, fault, transfer control, or fail to return.

Computed regions may alias each other, fixed locations, or storage used by dependencies. Consequently:

- repeated reads must not automatically be replaced by a cached value;
- repeated clears separated by calls remain distinct effects;
- an earlier write may be observed before a later overwrite;
- a captured scalar is distinct from a subsequent reread of its source memory.

For layout descriptions, a **selected-byte write** consists of an unsigned-byte selector read followed by a byte store at **selector value + 0x11**. The destination lies between 0x11 and 0x110 inclusive. It is not the selector’s address plus 0x11. “Selected” names this relationship only; no table ownership or object layout is established.

### 1. Entry and initial word propagation

The incoming word is first stored at **incoming base + 0x14**, before any dependency call.

F-000000000001 then receives the four arguments described above. Its target is calculated by adding 0xFFE to the incoming target-forming context value. A subsequent dependency chain yields a primary result; the incoming word is stored at **four times that result + 0x14**.

Further chains alternate calls with these word updates:

- four times a primary result is stored at **sixteen times that result + 0x14**;
- that captured scaled value is stored at **four times a later primary result + 0x14**;
- another four-times-scaled value is stored at its corresponding sixteen-times-scaled destination;
- zero is stored at **four times a subsequent primary result + 0x14**.

Calls between these writes receive scaled results, including repeated argument values. The recurring offset 0x14 does not establish that all destinations are instances of one record type.

### 2. Initial selectors and distinct two-argument calls

A later 32-bit primary result is captured as the **initial selector result**. Its low byte after multiplication by four is written through a selector read at:

**((initial selector result & 0x3FFFFFFF) >> 16) + 1**.

After intervening calls, another selected-byte write clears the destination selected at:

**((initial selector result & 0x3FFFFFFF) >> 17) + 1**.

A call receiving a newly obtained primary result shifted left by two follows that clear. A further chain obtains another primary result. **F-000000000006 then receives that primary result shifted left by two and a zero second argument.** The second argument derives from an explicitly zero scalar shifted left by 31. This call occurs **before** the following payload-byte read.

After one intervening call, the function reads an unsigned byte at:

**initial selector result << 3**.

F-000000000008 receives that byte’s zero-extended value identically in four visible argument positions. Another call intervenes before the selected-byte clear whose selector address is:

**((initial selector result & 0x3FFFFFFF) >> 7) + 1**.

**F-000000000007 follows this clear and receives a secondary-register value and zero.** Its first argument is not a scaled primary result. This is a distinct invocation from the earlier primary-result/zero call.

Further calls precede a selected-byte clear using the selector at absolute address one. A later chain culminates in F-000000000009, which receives:

- a later primary result shifted left by two;
- that same shifted result again;
- the earlier payload byte shifted left by 31.

Under 32-bit bit-vector arithmetic, the third argument is zero for an even payload byte and 0x80000000 for an odd payload byte.

Thus the critical order is: **primary-result/zero call; payload-byte read and four-position consumption; shift-by-seven selector clear; secondary-register/zero call**. These two two-argument calls must not be merged or exchanged.

### 3. Split results and mixed-width payloads

Subsequent dependency results include 64-bit values whose low portions are captured for later addressing and whose high portions are passed onward. Selected-byte clears occur between those captures and calls.

The following payload accesses occur in this relative order, with calls and selector operations interleaved:

- a word read at a captured low portion plus **((that portion >> 19) << 31)**;
- an 8-bit read at a secondary-register-derived base plus 0x0E;
- an unsigned-byte read at another captured low portion plus a secondary-register-derived displacement;
- a halfword read at a later secondary-register value shifted right by 19, plus the previously captured unsigned byte.

The first word-read adjustment is either zero or 0x80000000. It depends on bit 19 of the captured low portion, not on a general scaled index.

These payloads are supplied repeatedly in four visible argument positions. The 8-bit payload at offset 0x0E and the halfword have established access widths, but their ABI-level signed extension is not independently established.

A dependency also receives the same stack-relative address in all four visible positions. The addressed extent and initialization state are unknown.

Selectors in this portion derive from captured low portions, secondary-register values, and shifted or scaled quantities. Each selector byte is read at its own point in execution.

### 4. Offset-0x0D updates and scaled destinations

After the halfword payload is consumed, a split result supplies a captured low portion and a high portion. An earlier extracted value is stored as a word at **four times the high portion + 0x14**, then at another four-times-scaled destination obtained through later calls.

The captured low portion is also used as a byte-field base. At **base + 0x0D**, the function first stores the low byte of twice the base. After intervening dependency calls, it stores the constant four at that same address. Both writes remain observable.

This portion reads words at:

- **four times a captured primary result + 0x44**;
- **((that result & 0x3FFFFFFF) >> 20) + 0x44**.

Later word updates include:

- four times that captured result stored at **0x100 times a later result + 0x14**;
- **(a captured value & 0x03FFFFFF) >> 13** stored at a four-times-scaled destination plus 0x14;
- repeated propagation of four-times-scaled values through sixteen-times-scaled and later four-times-scaled destinations.

Calls use combinations of four-times scaling, 0x40-times scaling, high-bit shifts, repeated arguments, and explicit zeros.

### 5. Linked bases and offset-0x15 bytes

A captured result determines a sixteen-times-scaled region. The function clears a word at **region + 0x24**, then reads a word at **region + 0x74** and adopts the read word as a later base.

After more calls, a primary result shifted right by 20 is stored at **that base + 0x24**. A word read at **that base + 0x74** is passed to a dependency.

A selector read at **that base + 1** first selects a byte receiving the truncated value of an earlier masked result shifted right by 20. After intervening calls, the selector is read again for a zero-byte write. The selected destination need not remain the same.

Three later unsigned-byte reads occur at separately supplied secondary-register bases plus 0x15. Each is independently timed in the call sequence. The third byte is zero-extended and stored as a 32-bit word at **four times a later secondary-register value + 0x14**.

Another secondary-register-derived quantity is masked with 0x1FFFFFFF and shifted right by 18. That extracted value shifted left by 13 is passed to a dependency and later stored as a word.

### 6. Intermediate fixed-location clear and continued propagation

A fixed 32-bit location, designated **G0**, is explicitly cleared. This is an intermediate effect, not completion.

Subsequent behavior includes zero-word writes, scaled-result propagation, selector-mediated writes, and F-000000000003, a distinguished fixed-target invocation receiving the same captured 32-bit value twice.

Some scalar values in this later portion are explicitly assigned zero and subsequently only shifted, masked, or scaled before their next use. Their derived selector addresses therefore reduce to absolute address one under the stated arithmetic interpretation. This deduction concerns captured scalars; it does not establish that G0 remains zero.

One unsigned byte read from **a captured result + 1** is subsequently used as the base for another byte read at **byte value + 1**. The second read therefore addresses the range 1 through 256. It does not necessarily access the next byte of the original region.

### 7. Later fields, dependent loads, and fixed-location use

A byte-sized dependency result shifted left by two is written through a selector. Later operations include:

- a zero-byte write at **a captured, already four-times-scaled result multiplied by 0x2000, plus nine**;
- a byte write at **another captured base + nine**, containing that base’s low byte;
- a word read at **that base + 0x34**, passed alongside portions of a split result.

A dependent word-load chain begins at a secondary-register-derived base plus 0x14. A loaded word shifted right by 22 forms another address base. A subsequent loaded word is shifted right by one for a scaled write destination, while that word’s unshifted value also forms another read address. Word stores remain interleaved with these loads.

G0 is read during this chain. Its captured 32-bit value is shifted right by 23, producing at most 511. Shifting that extracted value right by 23 again necessarily produces zero, regardless of whether dependencies changed G0 after its clear.

Another fixed 32-bit location, **G1**, receives a full primary result before the next dependency invocation.

Several later updates read a word at **base + 0x74** and write its low byte at **base + 1**. The first such base is a captured unsigned byte read at absolute address one. Others are scaled dependency results. One scaled-base update is repeated after intervening calls, requiring a fresh word read.

A captured full word from one of these reads is also propagated into later word destinations. The representation additionally contains a word store at **(a captured scaled value & 0xFFFFFFFC) + 0x14**.

### 8. Final reads, composition, and terminal invocation

Near the end, G0 is read again. After intervening calls and writes, a word is read at **captured G0 + 0x14**. That captured word supplies payloads for later selected-byte writes: first its low byte, then the low byte of its value shifted right by 22.

Following further calls, F-000000000004 yields the last byte-sized result and an associated secondary pointer. The final visible memory effects occur in this order:

1. A zero byte is written through that secondary pointer.
2. A separate fixed-memory 24-bit quantity is combined with the byte result, and the resulting 32-bit word is stored in G1. The result supplies bits 0–7; the fixed-memory quantity supplies bits 8–31.
3. A zero halfword is written at **(an earlier captured low result & 0x3FFFFFFF) >> 7**.

The final halfword address has neither an added one nor selector indirection. Its range is zero through 0x7FFFFF, before accounting for the two-byte access extent. Alignment is not guaranteed.

F-000000000005 then invokes address zero with three visible zero arguments. The displayed return is reachable only if this invocation returns. No operating-system service, exception convention, or successful termination meaning is established.

## Side effects

- Direct writes affect bytes, halfwords, and words at computed and fixed locations.
- G0 is cleared and later reread; dependencies or aliasing stores may change it between those events.
- G1 receives a full primary result and later a separately composed full word.
- Fixed-memory symbols may overlap smaller symbols. Their disjointness from computed destinations is not established.
- Dependencies may have additional, unresolved effects.
- No explicit allocation, release, reference counting, or ownership transfer is established.
- Passing an address does not establish that a dependency owns, retains, validates, or releases its storage.
- The stack-relative address has no established retention permission or lifetime beyond the actual execution context.
- No rollback, transaction, synchronization, or atomicity guarantee is visible.
- Failure or nonreturn at any reached operation can leave an already modified state.

## Dependencies

These opaque IDs describe dependency roles or particular sites. They do not claim independently recovered implementations.

| Opaque ID | Visible role |
|---|---|
| F-000000000001 | Initial context-derived call; four input-related arguments |
| F-000000000002 | Group of other unresolved intermediate call sites |
| F-000000000003 | Distinguished fixed-target call receiving one captured value twice |
| F-000000000004 | Last byte-result site with an associated secondary pointer |
| F-000000000005 | Terminal address-zero invocation |
| F-000000000006 | Primary-result-shifted-left-two/zero call before the initial payload-byte read |
| F-000000000007 | Secondary-register/zero call after the shift-by-seven selector clear |
| F-000000000008 | Four-position consumption of the initial unsigned payload byte |
| F-000000000009 | Later three-argument call using two identical scaled results and that byte shifted left by 31 |

F-000000000002 groups distinct sites, not interchangeable callees. Their targets, argument counts, result uses, and ordering differ. Target calculations use several offsets, not one universal adjustment.

No dependency has an established global-preservation, pointer-validation, ownership, error, or normal-return contract. Empty displayed argument lists do not establish complete ABI-level independence from incoming register state.

## Acceptance examples

These are conditional static obligations, not executed tests.

1. **Entry order.** With a writable incoming base plus 0x14, the incoming word is stored there before F-000000000001 receives the input value in positions one, two, and four.

2. **Distinct call provenance.** If the primary result used by F-000000000006 is 0x12, its visible arguments are 0x48 and zero. If the later secondary-register value used by F-000000000007 is 0x3456, that call receives 0x3456 and zero—not a scaled primary result.

3. **Corrected intervening order.** The payload-byte read and F-000000000008 occur after F-000000000006. The shift-by-seven selector clear occurs after F-000000000008 and before F-000000000007.

4. **Selector interpretation.** A selector value of 0xFF selects destination 0x110, regardless of the selector’s own address.

5. **Fresh selector.** If an intervening dependency changes a selector byte, the later selected-byte write uses the later read value.

6. **Truncation.** A byte write with wider payload 0x123456AB writes 0xAB. That individual store does not write adjacent bytes.

7. **Parity extraction.** Initial payload bytes two and three produce third arguments zero and 0x80000000, respectively, at F-000000000009.

8. **Intermediate overwrite.** The doubled-base byte at offset 0x0D precedes the constant-four byte. A dependency between them can observe the earlier payload.

9. **Mutable G0.** A dependency changing G0 after its clear can affect a later G0 read. The twice-applied right shift by 23 of one captured 32-bit value still yields zero.

10. **Final composition.** If the last byte is 0x5A and the separately read 24-bit quantity is 0x123456, the final G1 assignment is 0x1234565A.

11. **Final address and failure.** An earlier captured low result of 0xFFFFFFFF gives final halfword address 0x7FFFFF. An alignment fault there prevents the terminal call. If execution instead reaches a nonexecutable address-zero target, normal return is not established.

## Uncertainties

The following remain **documented_only**:

- application purpose and intended successful outcome;
- actual ABI, complete public interface, and implicit-input provenance;
- dependency identities, implementations, clobbers, and ownership behavior;
- reliability and lifetime of recovered secondary-register values;
- argument extension for untyped narrow payloads;
- signed-overflow semantics and actual address-width behavior;
- endianness, alignment requirements, mappings, and alias relationships;
- storage extents, initialization, validity, ownership, and lifetimes;
- physical layout and overlap of fixed-memory quantities;
- runtime meaning and return behavior of the address-zero invocation;
- fidelity of the supplied representation to all machine-level effects.

## Evidence

Established observations derive solely from the supplied static representation: access widths, constants, masks, shifts, argument provenance, result splitting, memory-operation order, fixed-location assignments, and the absence of explicit branches or loops.

The early call ordering is supported by the placement of the primary-result/zero invocation before the payload-byte read, followed by four-position byte consumption, the shift-by-seven selector clear, and the distinct secondary-register/zero invocation.

Range and zero-result deductions are conditional on the stated 32-bit bit-vector interpretation. Structural descriptions do not establish application-level types or intent.

No runtime tests have been performed. No dependency implementations, execution trace, independent ABI evidence, or memory map were supplied.
