<!-- Trusted provenance: F-9a1effd92e78; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-9a1effd92e78

## Purpose

F-9a1effd92e78 repeatedly performs an ordered sequence of indirect calls, explicit-width memory accesses, a software breakpoint, and an adjustment to an implicit context base. The supplied representation contains no normal return path or termination condition.

Its application-level purpose is not established. The accesses do not, by themselves, establish initialization, allocation, copying between objects, device configuration, or event processing.

This contract describes the supplied representation’s observable behavior, conditional on dependencies returning, accesses completing, and execution resuming after the breakpoint. It does not guarantee completion of an iteration.

## Interface

### Entry and exit

- No explicit parameters are declared.
- No ordinary return value is produced.
- Execution repeats unconditionally after the final dependency call.
- A dependency, memory fault, or breakpoint may prevent further progress. No recovery path is shown.

### Implicit inputs

The operation depends on:

- An entry link-like value used to calculate the first indirect-call target.
- An implicit context base used for word accesses and subsequently decreased by 0xF4 bytes.
- An inherited address component used in a late byte-store destination.
- Two global word-sized cells, called the **primary-base cell** and **table-base cell** below.
- Memory reached through captured bases, dependency results, unsigned halfword values, and fixed addresses.
- Auxiliary result-register values associated with F-c006, F-c008, and F-c031.

These dependencies do not establish a supported public calling convention. In particular, the validity of auxiliary result-register values is unresolved.

### Widths and arithmetic

The representation distinguishes 8-bit bytes, 16-bit halfwords, 32-bit words, and 64-bit paired results. Each paired result is separated into low and high 32-bit halves.

Unsigned halfword reads are zero-extended. Halfword writes retain the source’s low 16 bits. Byte writes retain eight bits.

Unsigned low-word shifts have 32-bit results: shifting left discards bits above bit 31; shifting right by 26 yields a value from 0 through 63.

Some other calculations use signed-looking word operations, including:

- The sum passed to F-c006.
- The left shift of F-c019’s result.
- The six-bit left shift of F-c028’s low half.
- Multiplication of F-c028’s high half by four.
- Several address additions and the context-base subtraction.

The contract identifies their operands and widths without promising defined wraparound for signed overflow or invalid address arithmetic. Examples involving such operations either avoid overflow or state a bit-vector interpretation explicitly.

No byte order, alignment policy, aggregate layout, atomicity, or memory-ordering guarantee is established. Offsets below are byte offsets.

## Behavior

### Ordering and retained values

F-c001 through F-c037 identify call occurrences in execution order, not necessarily distinct targets. Established target equalities appear under **Dependencies**.

Dependencies may modify globals and reachable memory. A base or value already captured by this function remains the operand of subsequent operations in the supplied dataflow; a later global change does not substitute a new value for that capture. This does not guarantee that the captured address remains valid or that its memory contents remain unchanged.

The following phases describe one iteration.

### 1. Initial calls and primary-base writes

1. Call F-c001 with two explicit zero arguments. Its target is the current local link-like value plus 0xFFE.
2. Call F-c002, then F-c003.
3. Read the primary-base cell as a word and retain its value.
4. Call F-c004.
5. Call F-c005 and retain its paired result.
6. Call F-c006 with two word-sized arguments:
   - The pair’s low half.
   - Its high half plus its low half, with an additional byte-valued operand that has been set to zero.
7. Store the auxiliary result byte associated with F-c006 at captured primary base +0x11.
8. Call F-c007.
9. Store zero as a byte at captured primary base +0x15.

The primary-base cell is not reread between the two stores. F-c004 through F-c007 may change that cell, but such a change does not redirect either destination in the explicit dataflow.

### 2. Halfword store and paired-result distribution

1. Call F-c008. Store its halfword result at its auxiliary result-derived address +0x0C.
2. Call F-c009 and retain its paired result.
3. Store a zero halfword at the pair’s high half +0x2C.
4. Store the pair’s unsigned low half shifted right by 26 as a word at current context base +0x26C.
5. Call F-c010.
6. Read and retain a word-sized base from current context base +0x320.
7. Call F-c011, then F-c012.
8. Store the retained low half from F-c009, shifted left by two as an unsigned word, at captured base +0x54.

Both shifted values originate from the same F-c009 result. The base read from context +0x320 is captured before F-c011 and F-c012; those calls are not followed by a reload of that base.

### 3. Fixed-address read, breakpoint, and indexed store

1. Call F-c013, F-c014, F-c015, F-c016, and F-c017, in that order.
2. Read an unsigned halfword at absolute address 0x21C48994, which is 0x34 bytes beyond 0x21C48960.
3. Call F-c018.
4. Read a word at current context base +0x280 and pass it to F-c019.
5. Execute a software breakpoint with immediate value 0x92.

All remaining operations require continuation past the breakpoint.

6. Shift F-c019’s word-sized result left by two to obtain a destination displacement.
7. Derive 0x21C4 by shifting 0x21C48960 right by 18 and multiplying by four.
8. Read and retain a word-sized base at the zero-extended halfword from step 2 +0x44. This is the **retained destination base** used throughout the later phases.
9. Call F-c020.
10. Call F-c021 and retain its word result.
11. Read a word at the same zero-extended halfword +0x04. Call F-c022 with F-c021’s retained result and this newly read word.
12. Read and retain the table-base cell.
13. Call F-c023.
14. Store the captured table base’s low 16 bits at captured table base + the displacement derived from F-c019.

F-c019’s result selects the destination; it is not the value stored. F-c023 occurs after capture of the table base and before the store.

The halfword-derived accesses in this phase add offsets directly to the zero-extended halfword. No additional object or table base is supplied.

### 4. Fixed-derived base, paired result, and context adjustment

1. Call F-c024.
2. Read and retain a word-sized base at absolute address 0x2218, derived as 0x21C4 +0x54.
3. Call F-c025 and obtain an address-like result.
4. Store halfword 0x21C4 at that result +0x04.
5. Call F-c026, then F-c027.
6. Read an unsigned halfword at the base captured in step 2 +0x34. This is the **source selector** used below.
7. Call F-c028 and retain its paired result.
8. Store its low half as a word at current context base +0x258.
9. Read an unsigned halfword at its high half +0x14. This is the **publication value** used below.
10. Form two retained arithmetic values: the low half shifted left by six, and the high half multiplied by four.
11. Decrease the implicit context base by 0xF4.
12. Multiply the zero-extended source selector by four, then store F-c028’s low half as a word at that product +0x74.

The context-relative result store precedes the context adjustment and therefore uses the pre-adjustment base.

The final destination is derived directly from the source selector. There is no additional table-base lookup. Because the selector is unsigned and 16 bits wide, its fourfold product ranges from 0 through 0x3FFFC.

### 5. Mirrored word stores and base publication

1. Call F-c029.
2. Store the zero-extended publication value as a word at retained destination base +0x54.
3. Call F-c030.
4. Store the same retained publication value as a word at retained destination base +0x64.
5. Call F-c031.
6. Store the retained destination base’s low 16 bits at the sum of:
   - F-c028’s low half shifted left by six; and
   - The auxiliary address-like result associated with F-c031.
7. Call F-c032, then F-c033.

The publication halfword is read only once. F-c030 separates the two word stores, so they are not an indivisible update. A change to the original halfword source during F-c030 does not change the retained value used for the second store.

### 6. Byte transfer, final word access, and repetition

1. Read a byte at five times the zero-extended source selector. The address is formed by adding the selector to its previously formed fourfold product.
2. Call F-c034.
3. Store the retained byte at publication value + inherited address component +0x19.
4. Call F-c035.
5. Store F-c028’s retained high half multiplied by four as a word at retained destination base +0x44.
6. Call F-c036.
7. Assign 0x08712031 to the local link-like value.
8. Read a word at four times F-c028’s retained high half +0x44.
9. Call F-c037 with that newly read word.
10. Repeat from the start of the body.

The last word read is not a dereference through the retained destination base. Its address is computed directly from F-c028’s high half.

The link-like assignment occurs before the final argument read and F-c037. No subsequent assignment or reload changes that local value before the next F-c001 target calculation. Consequently, after a completed iteration, the next F-c001 target is fixed:

**0x08712031 +0xFFE = 0x0871302F.**

This is explicit dataflow in the supplied representation. It does not depend on whether F-c037 preserves a physical link register. Whether the representation accurately models the executable’s register behavior remains a separate uncertainty.

## Side effects

### Direct writes

Each iteration reaching every store performs the following ordered writes:

| Width | Destination | Content |
|---|---|---|
| 8 bits | Captured primary base +0x11 | Auxiliary byte after F-c006 |
| 8 bits | Captured primary base +0x15 | Zero |
| 16 bits | Auxiliary address after F-c008 +0x0C | F-c008 halfword result |
| 16 bits | F-c009 high half +0x2C | Zero |
| 32 bits | Context base +0x26C | F-c009 low half, unsigned right shift by 26 |
| 32 bits | Captured context-derived base +0x54 | F-c009 low half, unsigned left shift by two |
| 16 bits | Captured table base + indexed displacement | Low 16 bits of that base |
| 16 bits | F-c025 result +0x04 | 0x21C4 |
| 32 bits | Context base +0x258, before adjustment | F-c028 low half |
| 32 bits | Four times source selector +0x74 | F-c028 low half |
| 32 bits | Retained destination base +0x54 | Zero-extended publication value |
| 32 bits | Retained destination base +0x64 | Same publication value |
| 16 bits | Shifted F-c028 low half + auxiliary address after F-c031 | Low 16 bits of retained destination base |
| 8 bits | Publication value + inherited address component +0x19 | Previously captured source byte |
| 32 bits | Retained destination base +0x44 | Four times F-c028 high half |

Destinations and sources may overlap. No object separation, buffer extent, or non-aliasing guarantee is established. Earlier writes can affect later reads when their addresses coincide.

### State, ownership, and failures

The implicit context base decreases by 244 bytes once per iteration reaching that adjustment. The local link-like state becomes 0x08712031 near the end of such an iteration.

Dependencies may cause additional memory, global-state, ownership, or control-flow effects. No dependency is assumed to preserve globals, allocate memory, transfer ownership, or release memory.

No direct allocation or release operation is identified. Returned addresses are not proven writable, live, owned, or persistent.

There are no explicit null checks, bounds checks, status checks, cleanup paths, or error returns. The unconditional repetition is not an error-conditioned retry mechanism. Invalid accesses and failed arithmetic have environment-dependent consequences. A dependency may not return. The breakpoint may stop or transfer execution.

No rollback occurs: completed effects remain unless subsequent operations or external behavior change them.

## Dependencies

Occurrence-specific IDs describe interfaces visible at each call site:

| IDs | Visible role |
|---|---|
| F-c001 | Link-relative target; two explicit zero arguments |
| F-c002–F-c004 | No explicit arguments or consumed results shown |
| F-c005 | Paired result |
| F-c006 | Two arithmetic-derived word arguments; auxiliary byte subsequently used |
| F-c007 | Call between the primary-base byte stores |
| F-c008 | Halfword result and auxiliary address-like output |
| F-c009 | Paired result |
| F-c010–F-c018 | Calls interleaved with the specified accesses |
| F-c019 | Context-word argument; displacement-producing result |
| F-c020 | No explicit arguments or consumed result shown |
| F-c021 | Word result passed to F-c022 |
| F-c022 | Two word arguments |
| F-c023–F-c024 | Calls preceding the specified stores and reads |
| F-c025 | Address-like result |
| F-c026–F-c027 | Calls before the source-selector read |
| F-c028 | Paired result used through the remainder of the iteration |
| F-c029–F-c030 | Calls preceding the mirrored word stores |
| F-c031 | Auxiliary address-like output subsequently used |
| F-c032–F-c036 | Calls interleaved with final memory effects |
| F-c037 | Newly read word argument before repetition |

“No explicit arguments shown” does not establish an argument-free machine-level interface.

Fixed target arithmetic establishes these equalities:

| Call occurrences | Common calculated target |
|---|---|
| F-c005 and F-c006 | 0x08712DF9 |
| F-c009 and F-c013 | 0x08712E73 |
| F-c017 and F-c024 | 0x08712E89 |
| F-c019 and F-c031 | 0x08712EDD |
| F-c025 and F-c032 | 0x08712EBB |

These equalities are established, not unresolved aliasing questions. They do not establish identical arguments, results, or effects across occurrences. Implementations, semantics, and any additional runtime target resolution remain unknown.

## Acceptance examples

These are conditional static consequences, not executed tests. Each assumes the required accesses complete and relevant dependencies return.

1. **Captured primary base.** If the primary-base cell contains address A when captured, and F-c007 changes the cell to B without invalidating A, the following zero-byte store still targets A +0x15.

2. **Unsigned paired-result extraction.** If F-c009’s low half is 0xFC000003, the context store receives 63, and the later unsigned left-shifted word store receives 0xF000000C.

3. **Breakpoint stop.** If breakpoint 0x92 stops execution permanently, no subsequent operation in that iteration occurs. Earlier writes are not undone.

4. **Index versus stored content.** If F-c019 returns 3 and the captured table base is 0x12345678, the indexed destination is 0x12345684 and its stored halfword is 0x5678.

5. **Unsigned publication.** If the publication halfword is 0xFFFF, both mirrored word stores receive 0x0000FFFF, not 0xFFFFFFFF.

6. **Intervening observation.** If F-c030 reads the mirrored destinations, it may observe the +0x54 write before the +0x64 write. Changing the original publication-halfword location does not change the second store’s retained operand.

7. **Pre-adjustment context store.** If the context base is C before phase 4’s adjustment, F-c028’s low half is stored at C +0x258 before the base becomes C −0xF4, assuming that subtraction is valid.

8. **Captured byte.** If the source byte is 0xA7 when read, F-c034 may change its source location, but the following byte store still uses 0xA7.

9. **Next-iteration target.** If the iteration completes, the following F-c001 targets 0x0871302F, regardless of the first iteration’s entry link-like value. No result of F-c037 is assigned to that local state.

10. **Shared target, different occurrence.** F-c005 and F-c006 reach the same calculated target. The former supplies a paired result; the latter receives two explicit word arguments. A contract requiring different target addresses would contradict the supplied arithmetic.

## Uncertainties

The following remain **documented_only**:

- Application-level purpose and supported entry conditions.
- Actual ABI, register preservation, and auxiliary-output validity.
- Whether the implicit context base represents stack state or another mechanism.
- Dependency implementations, side effects, ownership behavior, and additional runtime target resolution.
- Address validity, memory lifetimes, alignment, byte order, device-memory behavior, concurrency, and atomicity.
- Signed-overflow behavior and address-boundary handling.
- Breakpoint handling and whether execution resumes.
- Fidelity of the represented loop, context adjustment, and local link-state model to the executable.

These uncertainties do not negate the representation’s explicit link-state assignment or the five established target equalities.

## Evidence

Established static evidence consists of the supplied function representation’s ordered calls, explicit-width accesses, retained values, arithmetic, fixed constants, breakpoint, context adjustment, local link-state assignment, and unconditional repetition.

The derived values 0x21C4, 0x2218, 0x0871302F, and the shared dependency targets follow from that arithmetic.

Descriptive terms such as “source selector” and “publication value” identify dataflow roles, not verified domain meanings. Higher-level intent remains unestablished.

No dependency implementations were inspected. No runtime execution or runtime tests have been performed.
