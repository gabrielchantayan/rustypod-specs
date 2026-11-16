<!-- Trusted provenance: F-4b0cfad1989a; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-4b0cfad1989a

## Purpose

F-4b0cfad1989a performs an ordered sequence of unresolved indirect invocations, transforms invocation-produced values, executes mixed-width memory updates, and reaches a final unresolved four-argument transfer.

No application-level purpose is established. Recurring offsets and character-valued constants do not establish text formatting, object initialization, protocol processing, or collection management.

This contract distinguishes:

- **Established static evidence:** explicitly represented operations, widths, value dependencies, and ordering.
- **Inferred intent:** possible interpretations that are not acceptance requirements.
- **documented_only:** behavior not resolved by the supplied representation or unavailable dependency contracts.

The effective interface includes hidden execution state and numerically derived addresses. The evidence is insufficient to define a safe standalone replacement without additional calling-convention and memory-environment information.

## Interface

### Explicit inputs

| Input | Width | Established use |
|---|---:|---|
| First argument | 32 bits | No explicit body-level use. |
| Second argument | 32 bits | No explicit body-level use. |
| Third argument | Unsigned 32 bits | Bits 10–19 are extracted and placed in bits 6–15 of an early invocation argument. |

The third-argument transformation produces 64 times the unsigned value of bits 10–19. Its range is 0 through 65,472, inclusive.

The absence of explicit uses of the first two arguments does not prove execution is independent of them. Unresolved invocation interfaces may involve hidden register inputs.

### Implicit inputs

The function also depends on:

- Incoming control context used to derive the first indirect target.
- Location-derived indirect targets.
- Primary results and secondary result channels from dependencies.
- Byte-sized and 32-bit global memory values.
- Contents of result-derived addresses.
- Actual address width, alignment rules, arithmetic semantics, and access permissions.

The global addresses below identify accesses represented in the supplied material. Their physical mapping and relationship to derived addresses remain **documented_only**.

### Outputs and completion

No explicit return value is represented. Observable outputs include dependency invocations, memory reads and writes, and the final transfer.

The final transfer receives, in order:

1. Constant 0x72.
2. A 32-bit value loaded from offset 0x64 of a late-loaded base.
3. A saved 32-bit value read earlier from offset 0x14 of the final retained base.
4. A later 32-bit value loaded from offset 0x74 of that same late-loaded base used for the second argument.

The second argument also determines the address of the last 32-bit store. The third argument is retained despite intervening writes; it is not reloaded at transfer time.

Ordinary return is represented after the transfer. Whether its target returns, faults, or transfers control elsewhere is **documented_only**.

### Preconditions

Complete execution requires executable indirect targets, meaningful result channels, and permitted memory accesses at every reached address. The environment must support the represented 8-, 16-, and 32-bit accesses, including potentially unaligned or low-numbered addresses.

No explicit validation enforces these conditions.

## Behavior

### Invocation phase

The initial phase is a long straight-line sequence of indirect invocations, with no explicit branch or loop.

The first target is incoming-context-derived; later targets are location-derived. Similar invocation patterns do not prove target identity.

Observable invocation behavior includes:

- Calls with no represented explicit arguments.
- Calls receiving shifted primary results.
- Calls receiving separately extracted lower and upper 32-bit components of represented 64-bit results.
- Calls combining primary results, secondary results, retained earlier values, and explicit zeros.
- Calls receiving the same transformed value in multiple argument positions.

Transformations include left shifts, unsigned right shifts, masks, and upper-component extraction. Mask widths include 24, 22, 21, 20, 19, 18, 17, 16, 14, 13, 12, and 11 bits. Some arguments select fields rather than merely preserving a masked value.

Argument order, explicit zeros, and retention across intervening invocations are meaningful. No dependency may be assumed to preserve globals merely because a result remains available for later use.

Some explicitly zeroed values undergo repeated shifts. Under fixed-width bit-vector semantics they remain zero; this does not establish an omitted counter or loop.

There is no visible status check, retry, error translation, or rollback.

### Width and arithmetic requirements

The representation distinguishes 8-, 16-, 32-, and 64-bit quantities.

For established unsigned 32-bit operations, right shifts are logical and results assigned to 32-bit destinations retain only representable bits. A later right shift acts on the already-truncated value, not on an unbounded intermediate.

Represented 64-bit results are sometimes split into independent 32-bit components. Shifting the lower component does not authorize shifting or carrying into the upper component.

Several shifts, multiplications, subtractions, and address calculations involve represented signed 32-bit quantities. Their exact machine behavior for overflow, negative operands, or excessive shift counts is **documented_only**. Unrestricted mathematical-integer semantics are not established.

Byte stores retain the low eight bits. Unsigned byte loads published through 32-bit stores are zero-extended. Access width must remain distinct from numeric value: storing 0x3a through an 8-bit access is not equivalent to storing it through a 32-bit access.

### Entry into direct memory updates

The direct-memory phase begins after the invocation phase. Its opening ordering includes:

- A 32-bit store at offset 0x14 from the upper component of the last paired result.
- A byte store at offset 9 from an earlier shifted base.
- A 32-bit scaled-value store at another result-derived base plus 0x54.
- An overwrite of the byte at offset 9.
- A 32-bit zero store at offset 0x14 of a large scaled base.
- A byte read at offset 5 of that scaled base.
- A zero store at offset 0x14 of another derived base.
- Zero-extending publication of the saved byte through a 32-bit store at offset 0x34 of the large scaled base.
- A 32-bit read at offset 0x14 of the large scaled base.

That final read is a read of then-current memory. It is not an entry-state read and cannot unconditionally be replaced with zero.

The second derived base comes from doubling an unsigned extracted quantity. Under 32-bit bit-vector interpretation, that quantity ranges from 0 to 508 in multiples of four, so the doubled base ranges from 0 to 1,016 in multiples of eight. This does not establish that the corresponding addresses are mapped or safe.

### Early global accesses and derived-memory dependencies

After another zero store and two identical byte stores at offset 0x15 of the doubled base, the function reads an unsigned byte from global address 0x05. That byte becomes an address base.

The next global effect is a byte write to global 0x01 containing the low byte of the doubled base. Between these global accesses and subsequent global effects, the function:

- Writes the extracted quantity’s low byte at offset 0x15 of the byte-derived base.
- Writes, then clears, a byte at offset 0x11 of the large scaled base, with an intervening derived 32-bit store.
- Reads and saves the 32-bit value at offset 0x54 of the byte-derived base, then overwrites that location.
- Publishes the extracted quantity at offset 0x34 of the doubled base.
- Follows offset-0x74 values and performs further 32-bit and byte writes.
- Twice writes the saved 32-bit value to its own derived offset-0x24 destination and its low byte to another base plus 9.
- Writes a dependency-derived shifted value at offset 0x14 of the doubled base.

The saved 32-bit value from the byte-derived base is then shifted left by one and written through a **32-bit global access at 0x54**.

After that global write, offset 0x14 of the doubled base receives the saved value twice. Offset 0x15 receives the low byte of four times that saved value twice. The function then writes the low byte of the doubled base to **global 0x15**, before continuing with derived stores at offsets 0x54 and 0x34.

These global accesses cannot be omitted or freely moved across surrounding derived accesses. No non-aliasing relationship is established.

### Dependent memory and overlapping accesses

The following region follows loaded values as address bases, scales saved values, and mixes byte and 32-bit accesses.

Established dependencies include:

- A byte read at offset 5 supplying the base for a 32-bit read at offset 0x74.
- Reads at offsets 4, 0x44, 0x64, and 0x74.
- Doubling or quadrupling loaded values to obtain later addresses.
- Self-related stores in which a base value is written to a destination derived from that base.
- Repeated byte reads separated by writes.

A byte-derived address is not evidence of an index into a conventional table. The representation also contains an expression with two reads of the same offset-0x74 location; it does not establish a single cached read or the relative evaluation order of those two reads.

Distinct derived bases may overlap. Byte writes may alter later wider reads, with the affected bits depending on endianness. No conventional structure layout, allocation size, or field ownership is established.

### Constant stores and repeated publication

A later region stores 0x3a through three widths: an 8-bit scaled-address store, a 16-bit differently scaled-address store, and a 32-bit store at a scaled base plus 0x20.

Another derived base is obtained by subtracting 150 from a previously established scaled quantity and multiplying the result by four. Its offset-0x20 destination receives four 32-bit publications in order:

- 0x20.
- 0x2f.
- The zero-extended byte read from offset 4 of that derived base.
- 0x20.

The intervening byte read remains observable.

**After those publications, the function loads the 32-bit global value retained as the publication base. Only then does it write 32-bit 0x20 to global 0x60.** The retained base therefore comes from the read before that global write, not from a subsequent reload. Independence of these accesses is not established.

The repeated publication region uses the retained base without visibly reloading its source global. Each publication group orders:

- A 32-bit payload store at a scaled destination plus 0x54.
- A byte store at retained publication base plus 0x19.
- A byte store at the adjusted derived base plus 0x11.
- A 32-bit scaled-base store at retained publication base plus 0x34.

The first group uses a scale based on the earlier saved value. Subsequent represented groups use a scale based on the subtraction-adjusted quantity. Their byte values are retained low bytes of the adjusted derived base and publication base.

The region is explicitly repeated without a visible loop condition. This contract does not numerically enumerate its multiplicity. Each represented repetition remains required; recurring stores must not be collapsed solely because their values recur.

### Final-region global sequence

For this region, **final retained base** means the doubled value established by the last relevant byte-directed offset-0x74 lookup. **Adjusted derived base** and **publication base** retain the meanings above.

The region begins with a **32-bit read from global 0xa9**, followed by a derived 32-bit read at offset 0x74. A byte is then written to **global 0x3a**. Its value is the low eight bits of eight times the subtraction-adjusted quantity plus 0x30, not necessarily a printable character.

Next, the function saves:

- The 32-bit value at final retained base plus 0x14.
- The 32-bit value at adjusted derived base plus 4.

Several derived writes follow: low-byte publications at offsets 0x0d and 9, byte constant 0x6c at adjusted derived base plus 8, and a 32-bit store of the final retained base at its offset 0x54.

Only after these writes does **global 0x67 receive the low byte of the saved global-0xa9 value**.

The later accesses to global 0x67 are separate reads:

| Global access | Required position relative to derived accesses |
|---|---|
| Read byte 0x67; write that byte to global 0x21 | After the 32-bit read at final retained base plus 0x44; before the byte write at adjusted derived base plus 0x10. |
| Read byte 0x67 for a byte store at adjusted derived base plus 9 | After the offset-0x40-to-offset-0x10 copy, the saved offset-4 publication to publication base plus 0x74, and the store to publication base plus 0x44. |
| Read byte 0x67 for another byte store at adjusted derived base plus 9 | After the byte write at final retained base plus 9, the saved global-0xa9 value’s 32-bit publication at adjusted derived base plus 0x44, and the 32-bit read at adjusted derived base plus 0x54. |

These reads must not unconditionally become copies of the low byte initially written to global 0x67. Intervening derived writes may alias that global. Each read observes memory at its represented position.

### Final copies, stores, and transfer

Between the global accesses above, the function publishes the value read from final retained base plus 0x44 through byte and 32-bit stores. A byte at final retained base plus 1 depends on a variable right shift of that represented signed value. The count is the low eight bits of the adjusted derived address. Machine behavior for negative operands or counts outside the supported range remains **documented_only**.

A 32-bit value read at adjusted derived base plus 0x40 is copied to that base plus 0x10. The earlier byte write at offset 0x10 precedes this wider overwrite.

After the last global-0x67 read and its associated byte store, the function writes a saved low byte to offset 0x0d of the base loaded from adjusted derived base plus 0x54. It then writes the saved value from **final retained base plus 0x14** to **final retained base plus 0x54**.

This is a **cross-offset copy**, not restoration of an original offset-0x54 value. The destination previously received the final retained base itself. Aliasing may affect actual memory relationships, but does not establish restoration.

Subsequent effects include:

- A 32-bit read at adjusted derived base plus 0x14.
- Further byte publications and an overwrite of adjusted derived base plus 0x44.
- A 32-bit read at final retained base plus 4, establishing the late-loaded base.
- A 16-bit store of 0x65 at twice that late-loaded base.
- A 32-bit read at late-loaded base plus 0x64.
- A byte publication of the saved adjusted-offset-0x14 value, then 32-bit constant 100 at another derived base plus 0x54.
- A 32-bit read at late-loaded base plus 0x74.
- A 32-bit read at final retained base plus 0x64.
- A byte store at final retained base plus 9.
- A 32-bit store of the saved late-offset-0x64 value at offset 0x54 from the final offset-0x64 result.
- A final 32-bit store, at twice the saved late-offset-0x64 value, containing the final offset-0x64 result.

The final transfer follows these effects. There is no visible commit marker or rollback boundary.

## Side effects

### Access layout

| Width | Established access coordinates |
|---|---|
| 8 bits | Globals 0x01, 0x05, 0x15, 0x21, 0x3a, 0x67; derived offsets including 1, 5, 8, 9, 0x0d, 0x10, 0x11, 0x15, 0x19. |
| 16 bits | Scaled-address constant stores of 0x3a and 0x65. |
| 32 bits | Globals 0x54, 0x60, 0xa9 and the publication-base source; derived offsets including 4, 0x10, 0x14, 0x20, 0x24, 0x34, 0x40, 0x44, 0x54, 0x64, 0x74. |

These are access coordinates, not structure declarations. Separation of global symbols or derived expressions does not establish disjoint storage.

### Ownership and errors

No explicit allocation, deallocation, or ownership transfer is visible. An address-like result is not proven newly allocated or owned. Dependencies may change memory, globals, resource state, or ownership.

No explicit error result or handler is represented. A non-returning invocation or fault can prevent later operations while leaving prior effects visible. Atomicity, recoverability, idempotence, and exception safety are not established.

## Dependencies

The following opaque placeholders identify unresolved dependency categories, not resolved shared targets:

| Opaque ID | Meaning |
|---|---|
| F-U001 | Initial incoming-context-derived invocation. |
| F-U002 | Location-derived indirect call sites; not one proven function. |
| F-U003 | Final unresolved four-argument transfer. |

No placeholder asserts purity, target uniqueness, memory preservation, or ownership behavior. Secondary result channels are required where consumed; missing ABI information does not authorize replacing them with zero or stale values.

## Acceptance examples

These are conditional static examples, not executed tests.

- **Third argument:** Inputs 0 and 1,023 produce transformed argument 0; 1,024 produces 64; 1,048,575 and 0xffffffff produce 65,472. This establishes only that argument transformation, not whole-execution equivalence.
- **Truncation:** Under unsigned 32-bit semantics, shifting 0x01000001 left eight bits retains 0x00000100. Shifting that result right eight yields 1.
- **Byte publication:** An unsigned loaded byte 0xfe becomes 32-bit 0x000000fe where zero-extending publication is represented, not 0xfffffffe.
- **Publication-base ordering:** If the source-global read yields one base and the following global-0x60 write affects that source through aliasing, publication still uses the already-retained base. Moving the read after the write is not equivalent.
- **Fresh global reads:** If an intervening derived write changes global 0x67 before a later read, the corresponding global-0x21 or derived-byte publication uses that later observed byte.
- **Cross-offset copy:** If final retained base plus 0x14 contains 0x11223344 when saved, and its offset 0x54 previously contains 0xaabbccdd, the later cross-offset store publishes 0x11223344. It does not restore 0xaabbccdd.
- **Ordered overwrites:** Observable offset-0x20 memory receives all four represented publications, including the intervening byte read; the final 0x20 alone is insufficient.
- **Interrupted execution:** A reached dependency that does not return does not require subsequent stores or the final transfer. No rollback of earlier effects is promised.

## Uncertainties

The following remain **documented_only**:

- Application purpose and dependency implementations.
- Actual indirect target identities, calling conventions, and hidden inputs.
- Whether paired results are genuine 64-bit returns or reconstructed result channels.
- Physical global layout, address mapping, permissions, and alignment behavior.
- Endianness, alias relationships, device behavior, and concurrent observation.
- Exact machine semantics of signed overflow and problematic shifts.
- Final-target return behavior and dependency ownership effects.

The complete call-site inventory and numeric repeated-publication count are not enumerated here. Their omission is not permission to change represented multiplicity or order.

“Publication” is descriptive terminology, not proof of a synchronization protocol. Initialization or text-generation intent remains inference only.

## Evidence

This contract uses the supplied static representation, independently of the prior draft’s conclusions.

Established evidence includes the argument widths and extraction, indirect invocations, result transformations, mixed-width accesses, explicit global-access sequence, read-before-write publication-base relationship, fresh reads of global 0x67, cross-offset 0x14-to-0x54 copy, and final four-argument transfer.

No dependency implementations, instruction-level validation, ABI documentation, memory map, or runtime observations were available. No runtime tests have been performed.
