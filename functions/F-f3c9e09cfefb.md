<!-- Trusted provenance: F-f3c9e09cfefb; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-f3c9e09cfefb

## Purpose

F-f3c9e09cfefb performs descriptor-directed memory updates, invokes opaque dependencies, and terminates its represented ordinary paths at invalid-instruction boundaries. No ordinary return value or recoverable error result is established.

A retained descriptor selector distinguishes two behaviors:

- **Zero selector:** exceptional processing or construction of a 16-byte region, an auxiliary update, and a consumer invocation.
- **Nonzero selector:** two halfword updates and a lookup, followed by an invalid-instruction boundary, construction of a 28-byte region, or an alternate consumer invocation.

These are observable memory and control-flow behaviors, not established application-level operations. The regions resemble structured records, but their format, purpose, ownership, and discovery by consumers are unknown. Neither region’s destination is an explicit argument to its subsequent consumer.

This contract describes static evidence only. Faults, exceptional transfers, and dependencies that do not return can prevent later effects.

## Interface

### Explicit inputs and outputs

| Input | Representation | Established use |
|---|---|---|
| First opaque input | 32 bits | No direct use established |
| Second opaque input | 32 bits | No direct use established |
| Numeric input | Signed 32-bit integer | Byte payload, address calculations, and compact-consumer argument |
| Descriptor address | 32-bit address representation | Configuration reads and an initial byte write |

The first two inputs have no established direct use. This does not prove that their incoming execution state is invisible to an invoked dependency.

There is no established output parameter, ordinary return value, or returned status. Results obtained from dependencies are used internally or supplied to later dependencies.

### Descriptor layout

Offsets are measured in bytes. A word is 32 bits; a halfword is 16 bits.

| Offset | Access | Role |
|---|---|---|
| 0x00 | Word read | Initial dependency argument |
| 0x04 | Word read | Contribution to the preliminary byte destination |
| 0x08 | Word read | Zero/nonzero selector |
| 0x0C | Word read | Preliminary byte payload and compact auxiliary base |
| 0x10 | Word read | Compact field, lookup base, and larger-region destination |
| 0x14 | Word read | Contribution to the first halfword destination |
| 0x1D | Byte write | Low eight bits of the numeric input |

This access map does not establish a complete object type, allocation size, or ownership relationship.

### Implicit inputs

Behavior also depends on:

- An independent signed 32-bit operand used to determine subtraction overflow.
- Incoming linkage information used to form the initial dependency target.
- Exception-state storage.
- A shared 32-bit data word.
- A shared control cell whose complete type is unresolved.
- Frame-associated storage and addresses.

The **anchor** is the frame-associated address embedded in the larger region.

| Location relative to anchor | Established role |
|---|---|
| −0xB8 | Signed 32-bit source for the first halfword update |
| −0x0C | Signed 32-bit base for the second halfword destination |
| −0x08 | 32-bit source for a larger-region field |
| −0x04 | Source of a packed field’s upper 24 bits |
| +0x140 | Address passed to the larger-region consumer |

The anchor and its surrounding storage are not established as separately allocated outputs. Their lifetime beyond this invocation is unknown.

### Preconditions and representation limits

Each reached access requires accessible storage for its full width. Each reached invocation requires a usable target. No destination validation, bounds checking, allocation check, or recovery mechanism is established.

Byte and halfword stores retain the low eight and sixteen bits, respectively. Stored address representations occupy 32 bits. Two dependencies provide represented 64-bit results.

Only the lookup-address addition has established unsigned 32-bit modulo-2³² behavior. Other calculations must not be treated as wrapping merely because their operands or destinations have 32-bit representations.

Byte order, alignment requirements, atomicity, and physical memory-transaction counts are **documented_only**.

## Behavior

### Common observations and effects

An overflow decision concerns subtraction of 45 from the independent signed 32-bit operand. It is made before the first represented write and does not depend on the explicit numeric input.

The common effects have the following established order:

| Earlier event | Later event |
|---|---|
| Write the numeric input’s low byte at descriptor offset 0x1D | All descriptor reads |
| Read descriptor offsets 0x00, 0x08, 0x0C, 0x10, and 0x14, in that order | Read descriptor offset 0x04 |
| Read descriptor offset 0x04 | Perform the preliminary byte write |
| Perform the preliminary byte write | Invoke F-0d1b65e7a930 |

The preliminary byte destination combines the numeric input with the offset-0x04 observation. Its payload is the low byte of the retained offset-0x0C value. Complete destination arithmetic and address-conversion semantics remain **documented_only**.

F-0d1b65e7a930 receives the retained offset-0x00 word. Its target is linkage-derived and involves a displacement of 0x950; complete target-formation semantics are unresolved. If it returns, it supplies a represented 64-bit result.

The selector and other configuration values observed before this invocation remain retained. Dependency changes to descriptor storage do not refresh them.

Aliasing still matters. The initial descriptor byte write can affect a subsequent descriptor observation. The preliminary write can affect memory accessed later, but cannot revise observations already retained.

### Zero-selector behavior

A retained selector of zero selects either exceptional processing or compact-region construction.

#### Exceptional processing

The independent subtraction-overflow decision is true exactly when its operand is in the inclusive range −2147483648 through −2147483604.

For that range, common processing still occurs first, including both byte writes and the initial dependency invocation. The exceptional sequence then has this represented order:

1. Assignment of zero to exception-state storage.
2. Software interrupt request with selector 0xE2.
3. Undefined-instruction operation with selector 0x94 and context scalar 0x0898412C.
4. Exception-derived invocation F-a58f9ae3d291.

The zero assignment’s width is unknown. The context scalar does not establish a memory access at that address.

No ordinary continuation from this sequence into compact construction is established. Whether a handler redirects execution, resumes execution, or supplies a usable invocation target remains **documented_only**.

#### Compact-region construction

When the overflow decision is false, the compact destination is formed from the signed numeric input shifted right by 24 positions and converted to an address.

For nonnegative inputs, the resulting destination is between addresses 0 and 127 inclusive. Negative signed-shift behavior and conversion of a negative result to an address remain unresolved. An arithmetic right shift must not be assumed.

The 16-byte region has this layout:

| Offset | Width | Contents |
|---|---:|---|
| 0x00 | 64 bits | Result from F-0d1b65e7a930 |
| 0x08 | 32 bits | Retained descriptor offset-0x10 value |
| 0x0C | 32 bits | Compact destination address |

Field writes occur in increasing offset order.

After these fields, an auxiliary 32-bit write places the retained offset-0x10 value at the retained offset-0x0C base plus 0x38 bytes. The base is interpreted as signed 32 bits; out-of-range arithmetic is unresolved.

The auxiliary destination need not be disjoint from the compact region. Its later write can replace part of the newly constructed contents.

F-746ee9a302d1 is then invoked with:

- The initial dependency result’s low 32 bits.
- Its high 32 bits.
- The numeric input minus 44.

The compact destination is not an explicit argument.

Subtraction by 44 is representable as signed 32-bit arithmetic exactly when the numeric input is at least −2147483604. The independent overflow decision does not validate this subtraction: it concerns another operand and subtraction by 45. Signed conversion of a result half with its top bit set is also unresolved.

A normal return from F-746ee9a302d1 reaches an invalid-instruction boundary.

### Nonzero-selector behavior

Every nonzero retained selector selects this behavior. The initial dependency still runs, but its result has no established direct consumer on this path.

#### Preparation and lookup

The first halfword update uses the signed source at anchor −0xB8. Its destination combines that source with the retained descriptor offset-0x14 value; its payload is the source’s low sixteen bits. Out-of-range destination arithmetic is unresolved.

The shared 32-bit data word is captured after this update and before a software breakpoint with selector 0xD9. An overlapping halfword write can therefore affect the captured value. Breakpoint-handler mutation does not replace that retained capture.

After the breakpoint, the shared control cell participates in three distinct observations:

| Observation | Retained consequence |
|---|---|
| Comparison against 74 | Greater-than decision |
| Subtraction of 75 | Represented 32-bit difference and its zero/nonzero decision |
| Interpretation as an address | Base for an unsigned halfword read at byte offset 0x2E |

These observations are not established as one stable snapshot. Complete cell typing, comparison signedness, subtraction semantics, and address conversion remain **documented_only**. This logical observation sequence does not establish a physical read count.

The unsigned halfword supplies a byte displacement. It is not scaled as an element index. Addition to the retained descriptor offset-0x10 value uses unsigned 32-bit arithmetic modulo 2³². A 32-bit lookup value is read at the resulting address.

After that lookup, the second halfword update writes the anchor address’s low sixteen bits at the signed base observed at anchor −0x0C plus 0x2E bytes. Out-of-range arithmetic is unresolved.

The lookup and second halfword write both precede ordinary outcome selection, including when the retained difference is zero.

#### Outcome partition

| Retained difference | Lookup value | Ordinary outcome |
|---|---|---|
| Zero | Any | Invalid-instruction boundary after preparation |
| Nonzero | Zero | Larger-region processing |
| Nonzero | Nonzero | F-6ca8413df720, then an invalid-instruction boundary if it returns |

The retained greater-than decision does not change this partition.

#### Larger-region processing

F-b509fc1a24e8 is invoked without explicit arguments. If it returns, its represented 64-bit result contributes to a 28-byte region beginning at the retained descriptor offset-0x10 address.

| Offset | Width | Contents |
|---|---:|---|
| 0x00 | 64 bits | Result from F-b509fc1a24e8 |
| 0x08 | 32 bits | Shared data captured before the breakpoint |
| 0x0C | 32 bits | Value observed at anchor −0x08 |
| 0x10 | 32 bits | Larger-region destination address |
| 0x14 | 32 bits | Upper 24 bits from anchor −0x04; low byte from the retained difference |
| 0x18 | 32 bits | Anchor address |

Field writes occur in increasing offset order.

The frame-associated field sources are not established as snapshots taken before F-b509fc1a24e8. The source for offset 0x0C is consumed after writes at offsets 0x00 and 0x08. The source supplying the packed field’s upper bits is consumed after the write at offset 0x10. Dependency effects and overlapping earlier writes can therefore affect these observations.

The packed field does not necessarily preserve the destination’s previous upper bits: its upper 24 bits come from frame-associated storage. Only this field truncates the difference to eight bits. Outcome selection uses the full retained zero/nonzero decision.

F-2a497c08bd63 receives the new dependency result’s low and high 32-bit halves, in that order, followed by anchor +0x140. Its third argument is not the larger-region destination.

A normal return reaches an invalid-instruction boundary.

#### Alternate consumer and excluded dependency

When both the retained difference and lookup value are nonzero, F-6ca8413df720 is invoked without explicit arguments. Its represented 32-bit result has no ordinary reachable consumer.

F-4747de84c84c has no ordinary reachable invocation:

- A false retained greater-than decision leads to an invalid-instruction boundary after the alternate consumer.
- A true retained greater-than decision, combined with the already-required nonzero difference, also reaches an invalid-instruction boundary before F-4747de84c84c.

This exclusion depends on retained decisions, not on preservation of the shared control cell. Reads needed only to prepare arguments for the excluded invocation are likewise not ordinary reachable effects.

## Side effects

Conditional on reaching the relevant portion, direct writes are:

| Portion | Writes |
|---|---|
| Common processing | Two byte writes |
| Exceptional processing | Zero assignment of unresolved width |
| Compact processing | One 64-bit write and three 32-bit writes, including the auxiliary update |
| Nonzero-selector preparation | Two halfword writes |
| Larger-region construction | One 64-bit write and five 32-bit writes |

Descriptor storage, constructed regions, auxiliary destinations, frame-associated storage, shared cells, and exception-state storage are not established as disjoint.

Aliasing can alter later observations or overwrite earlier output. It cannot retroactively change retained values or decisions. Dependencies and handlers may add effects, including mutation of completed regions.

No direct allocation, release, reference-count operation, ownership transfer, synchronization, or cleanup is established. Dependency behavior in these areas is unknown.

Effects are not transactional. A fault or nonreturning operation may leave a completed prefix without rollback. Access widths do not establish atomic completion under faults.

## Dependencies

Opaque IDs identify invocation boundaries, not semantic roles or preservation guarantees.

| ID | Established interface or reachability |
|---|---|
| F-0d1b65e7a930 | Linkage-derived target; one 32-bit argument; represented 64-bit result |
| F-a58f9ae3d291 | Exception-derived target; no explicit arguments; continuation unresolved |
| F-746ee9a302d1 | Initial result halves, low first, followed by numeric input minus 44 |
| F-b509fc1a24e8 | No explicit arguments; represented 64-bit result |
| F-2a497c08bd63 | Larger-path result halves, low first, followed by anchor +0x140 |
| F-6ca8413df720 | No explicit arguments; represented 32-bit result |
| F-4747de84c84c | No ordinary reachable invocation |

“No explicit arguments” does not mean “no inputs.” Dependencies may observe accessible memory and implicit execution state.

No dependency is assumed to preserve globals, frame-associated sources, or constructed regions; return normally; retain storage; acquire ownership; or release storage. Result widths establish neither ownership nor an error convention.

## Acceptance examples

These are conditional static acceptance criteria, not executed tests. Each assumes accessible reached storage, usable targets, continuation through preceding operations, and defined arithmetic where required.

| Conditions | Required consequence |
|---|---|
| Numeric input is 0x123456AB; retained descriptor offset-0x0C value is 0x102030CD | Descriptor offset 0x1D receives 0xAB; the preliminary destination receives 0xCD. |
| Retained selector is zero; the initial dependency subsequently changes descriptor offset 0x08 to seven | Zero-selector behavior remains selected. |
| Zero selector; independent operand is −2147483604 | Common effects precede exceptional processing. At −2147483603, the overflow decision is false. |
| Only the explicit numeric input changes | The independent overflow decision is unchanged. |
| Compact construction is reached with numeric input 0x40000000 and retained auxiliary base 20 | The compact destination is address 64. The auxiliary write at address 76 subsequently replaces its final word. |
| Numeric input is −2147483604 | Subtraction by 44 yields −2147483648. An input one smaller requires an out-of-range signed result. Negative compact-address formation remains unresolved. |
| Lookup base is 0xFFFFFFE0; halfword displacement is 0x0040 | The lookup address is 0x00000020. The displacement contributes 64 bytes without scaling. |
| Retained difference is zero | The lookup read and second halfword write still occur before the invalid-instruction boundary. |
| Comparison observes 74; subtraction observes 76; lookup value is nonzero | If the unresolved cell semantics yield difference one, F-6ca8413df720 is selected despite the false greater-than decision. |
| Retained difference is 256; lookup value is zero; packed-field source is 0x76543210 | Larger-region processing occurs; offset 0x14 receives 0x76543200. |
| Larger-region offset 0x08 overlaps the source at anchor −0x08 | That earlier write can affect the value subsequently consumed for offset 0x0C. |
| F-6ca8413df720 changes the shared control cell | The mutation does not make F-4747de84c84c ordinarily reachable. |

## Uncertainties

The following remain **documented_only**:

- Exception-state assignment width, handler behavior, exceptional redirection, target validity, and possible resumption.
- Dependency implementations, implicit inputs, preservation guarantees, ownership behavior, and error conventions.
- Complete shared-control-cell typing, comparison signedness, subtraction semantics, and address interpretation.
- Negative signed-right-shift behavior and conversion of negative results to addresses.
- Complete preliminary-destination arithmetic and linkage-derived target formation.
- Out-of-range arithmetic other than the explicitly unsigned lookup-address addition.
- Signed conversion of result halves whose top bit is set.
- Byte order, alignment, atomicity, and physical transaction counts.
- Storage accessibility and lifetime beyond the stated preconditions.
- Whether dependencies discover, interpret, mutate, retain, or release either region.
- Behavior beyond represented invalid-instruction boundaries.

These unknowns do not justify assuming stable globals, preserved output, ownership transfer, recoverable termination, or cleanup.

## Evidence

**Established static evidence:** The supplied representation supports the access widths, descriptor observation order, retained configuration, independent overflow operand, invocation boundaries, region layouts, and alias-sensitive source timing described here. It also establishes unsigned wrapping for the lookup-address addition and places the lookup and second halfword update before outcome selection.

The exclusion of F-4747de84c84c follows from the retained branch decisions: ordinary arrival at the alternate consumer already requires a nonzero difference, and either value of the retained greater-than decision reaches an invalid-instruction boundary before the excluded invocation. No global-preservation assumption is needed.

**Inferred intent:** The grouped writes appear to construct records for later processing. This interpretation establishes no application-level format, discovery mechanism, ownership transfer, or persistence requirement.

**Unknowns:** Complete storage declarations, dependency implementations, exceptional-handler contracts, and several arithmetic interpretations are unavailable. Prior prose, comments, and reviewer feedback are not independent behavioral evidence.

No runtime tests have been performed.
