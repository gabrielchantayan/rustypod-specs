<!-- Trusted provenance: F-6df2e1cea3ea; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-6df2e1cea3ea

## Purpose

F-6df2e1cea3ea executes an ordered sequence of unresolved indirect calls, transforms explicit and implicit 32-bit inputs, mutates memory through computed and fixed addresses, and reaches a final unresolved indirect transfer.

The supplied static representation does not establish an application-level purpose. Character-valued constants, repeated field offsets, and self-address stores do not establish formatting, initialization, allocation, serialization, or container maintenance.

This contract distinguishes:

- **Established static evidence:** accesses, transformations, and ordering explicitly represented in the supplied body.
- **[INFERENCE]:** arithmetic consequences conditional on faithful recovery and 32-bit bit-vector execution.
- **documented_only:** interfaces or execution properties that the supplied material does not resolve.

No runtime tests have been performed. Acceptance examples are conditional requirements, not observed execution results.

## Interface

### Explicit inputs

The represented interface has four arguments and no declared return value.

| Argument | Width and interpretation | Visible contribution |
|---|---|---|
| First | 32-bit opaque value | No explicit use in the body. |
| Second | 32-bit opaque value | No explicit use in the body. |
| Third | Unsigned 32-bit value | Upper eight bits and, separately, upper nine bits enter early calls. |
| Fourth | Signed 32-bit value | Its seven-position left shift is supplied twice to one early call. |

The first two arguments are not established as irrelevant to execution. Unresolved callees may consume incoming machine state beyond the arguments displayed at individual call sites.

The third argument is not visibly dereferenced. The fourth is transformed before its visible use as a call argument.

### Implicit inputs

Behavior also depends on incoming control state, two additional unsigned register values, primary and secondary call results, fixed storage, existing computed-address contents, and an unresolved final target.

Some returned quantities are represented as 64-bit results divided into low and high 32-bit words. Other consumed secondary words have no complete producer interface in the supplied material.

Consequently, the four explicit arguments do not determine the access trace or final transfer arguments.

### Outputs and completion

There is no established result buffer, ordinary return value, or success status. Observable outputs comprise dependency effects, direct memory accesses, and the final transfer.

Its four displayed arguments are:

1. `0x72`.
2. A word loaded late from offset `0x64` of a late-discovered base.
3. A word saved earlier from offset `0x14` of another computed base.
4. A later word loaded from offset `0x74` of the late-discovered base.

A return is represented after this transfer. Whether that represents an ordinary call-and-return sequence is **documented_only**.

### Preconditions

Reaching completion requires executable indirect targets, compatible calling conventions, meaningful implicit inputs, and accessible storage at every reached address and access width.

The actual platform must support any low-address, overlapping, or unaligned accesses encountered. Arithmetic and shifts must follow the intended target-machine semantics. No visible validation establishes these conditions.

## Behavior

### Indirect calls and field extraction

A long prefix consists of indirect calls interleaved with arithmetic. No explicit data-memory store appears in this prefix. Callees may nevertheless read or mutate memory, alter globals, allocate or release storage, fault, or fail to return.

The first target depends on incoming control state plus `0xFFE`. Later targets predominantly use changing code-associated values with the same displacement. Repeated construction does not establish one shared dependency identity.

Calls display between zero and four arguments. Absence of displayed arguments does not prove absence of consumed register or stack inputs. Calls remain ordered even when their results are not visibly used.

Established transformations include:

| Input material | Extraction or transformation |
|---|---|
| One implicit unsigned word | Bits 22–31, then a 13-position left shift |
| Third explicit argument | Bits 24–31, and separately bits 23–31 |
| Fourth explicit argument | Seven-position left shift, duplicated at one call |
| Another implicit unsigned word | Eleven-position logical right shift |
| Returned word | Low 11 bits, then a ten-position left shift |
| Returned word | Bits 4–23, then a 13-position left shift |
| Same returned word | Bits 19–23, later used through shifts and its low bit |
| Saved high result word | Bits 11–17, then a 14-position left shift |
| Several result words | Bits 8–23 |
| Another saved high word | Bits 4–16 |
| Returned word | Bits 12–23 |

Numerous low result words are shifted left eight positions. Paired results supply their low and high words separately. Explicit zero arguments occur at particular positions and must not be omitted from the represented call interfaces.

The third argument’s nine-bit contributions precede replacement of its working value with its upper byte. Later shifts of that working value therefore operate on the extracted byte, not the original argument.

### Width-sensitive arithmetic

**[INFERENCE]** Under 32-bit bit-vector semantics, left shifts truncate beyond bit 31. The extracted 20-bit field shifted left 13 therefore retains only its lower 19 bits.

Other conditional consequences include:

- A value shifted left 31 and subsequently left 30 becomes zero.
- A value shifted left 14 and subsequently left 23 becomes zero.
- A word shifted left 16 has zero low 16 bits; masking those bits and shifting right five produces zero.
- Explicitly zeroed working values remain zero through subsequent represented shifts until reassigned.

These consequences do not authorize deleting adjacent calls: their independent effects are unknown.

Signed shifts and multiplications also appear. Portable signed-arithmetic behavior is not established. In particular, the contract does not replace target arithmetic with an assumption that every represented signed expression is defined by a high-level language.

### Entry into direct memory mutation

The direct-memory region uses independently derived address material:

- A returned low word shifted left 25.
- Another returned word multiplied by `0x02000000`.
- A compact unsigned quantity obtained by an eight-position left shift, masking with `0x7FFF`, and a six-position right shift.
- Twice that compact quantity.
- Both words of the final paired prefix result.

These are address derivations, not validated object references.

**[INFERENCE]** With 32-bit bit-vector arithmetic, the compact quantity is a multiple of four from 0 through 508. Its doubled address is from 0 through 1016. The larger shifted bases may also be zero.

The opening ordering is significant. A word store at offset `0x14` through the final high result precedes a byte store at offset `0x09` through the 25-position-shifted base. A word store through the low result scaled by 256 follows, then the same byte destination is written again with a different source.

Next, the multiplication-derived base has its offset-`0x14` word cleared before an explicitly byte-sized read at offset `0x05`. The doubled compact base receives a word clear at offset `0x14`; the previously captured byte is zero-extended into a word at offset `0x34` of the multiplication-derived base. Its offset-`0x14` word is then read back. Another clear through the doubled compact base precedes two byte stores at its offset `0x15`.

Aliasing can affect these reads. A clear does not guarantee a later zero result if an intervening write reaches overlapping storage.

### The unresolved-width fixed read near `0x05`

After those initial duplicate byte stores, the body reads symbolic fixed storage associated with address `0x05` and converts its value to unsigned 32 bits. The supplied material gives no declaration establishing the storage access width.

This is **not established as an eight-bit read**. Conversion to 32 bits establishes the resulting value’s width, not the original memory access width or extension behavior.

The resulting unsigned value becomes a computed base. Its acquisition precedes a fixed byte write at `0x01`, followed by computed byte and word accesses using that base and other saved address material.

This unresolved-width global access is distinct from the explicitly byte-sized reads through computed bases at offset `0x05`. Those computed reads do establish eight-bit acquisition followed by zero-extension where converted to a word.

### Field replacement and pointer chasing

The middle region uses byte tags, word replacement, address scaling, self-address stores, and pointer chasing.

| Access width | Representative relative offsets |
|---|---|
| 8 bits | `0x01`, `0x05`, `0x08`, `0x09`, `0x0D`, `0x10`, `0x11`, `0x15`, `0x19` |
| 16 bits | Separately doubled or otherwise scaled addresses |
| 32 bits | `0x04`, `0x10`, `0x14`, `0x20`, `0x24`, `0x34`, `0x40`, `0x44`, `0x54`, `0x64`, `0x74` |

This inventory does not define a common structure layout. Identical offsets occur through unrelated bases.

Words loaded from several of these offsets become addresses or scaled address material. Explicit byte reads at offset `0x05` can select a subsequent word-read address at offset `0x74`.

Repeated self-address and equal-value stores remain part of the represented trace. Their multiplicity may matter for overlapping storage, device-backed memory, or external observation. No ownership or linked-structure protocol follows from these stores.

Fixed storage also participates during this region: a word at `0x54` receives a shifted computed value, and a later byte write targets `0x15`. Computed mutations occur between these fixed accesses; they must not be treated as a separate, nonaliasing namespace.

### Constants and repeated publication

A later region writes `0x3A` at byte, halfword, and word widths through separately scaled addresses.

Another computed word location, at offset `0x20`, receives four ordered values:

1. `0x20`.
2. `0x2F`.
3. A zero-extended byte read from offset `0x04` of the same computed base.
4. `0x20` again.

The intervening read and intermediate stores are observable parts of the behavior.

The publication base is then loaded as a 32-bit word from separate global storage. **That load precedes the fixed word store of `0x20` at `0x60`.** Publication subsequently uses the captured base; the representation does not reload it after the store at `0x60`. If these storage locations overlap, the captured base is still the value obtained before that store.

A finite straight-line region repeats four-access publication groups:

- A word store at offset `0x54` of a scaled base.
- A byte store at offset `0x19` of the captured publication base.
- A byte store at offset `0x11` of another computed base.
- A word store at offset `0x34` of the captured publication base.

The first group uses a different scaled base for the first destination and fourth stored value than subsequent groups. Byte values are truncations of already computed addresses, not fresh global loads.

Every represented occurrence is significant. There is no visible input-controlled loop or authorization to collapse the region to its final stores. Numerical repetition multiplicity is **documented_only** in this contract.

### Ordered late captures

For the late region, the **field base** denotes the computed base used by the four offset-`0x20` stores above. The **linked base** denotes a previously derived doubled pointer. These are descriptive labels, not object-type claims.

Immediately after repeated publication:

1. The 32-bit word at fixed address `0xA9` is captured.
2. A computed word at offset `0x74` of the published-value base is captured.
3. Fixed address `0x3A` receives a byte derived from the adjusted publication address: the low byte of eight times that adjusted value plus `0x30`.
4. The word at linked-base offset `0x14` is saved.
5. The word at field-base offset `0x04` is saved.

The fixed byte write therefore separates the first two captures from the latter two. If it overlaps either later word read, that read observes the ordered memory state, subject to target byte layout. Moving all captures before the write would change the contract.

Subsequent byte mutations include field-base offsets `0x08`, `0x0D`, and `0x09`; offset `0x08` receives `0x6C`. A linked-base offset-`0x54` word receives that base’s own address.

### Fixed byte `0x67` and its separate reads

The low byte of the previously captured `0xA9` word is assigned to fixed address `0x67`. The body then reads the linked-base word at offset `0x44` before reading byte `0x67` for assignment to fixed byte `0x21`.

Later, two computed byte stores explicitly reread byte `0x67`, both targeting field-base offset `0x09` at different points:

- The first follows several computed mutations, a field-base offset-`0x40` word capture, replacement of the field-base offset-`0x10` word, and a publication-base offset-`0x44` store.
- Between the first and second rereads, the body writes linked-base offset `0x09`, writes the captured `0xA9` word at field-base offset `0x44`, and reads the field-base word at offset `0x54`.

These are separate reads, not reuse of the initially assigned byte. Intervening computed writes may overlap `0x67`; therefore the assignment to `0x21` and the two computed stores need not receive equal bytes.

The supplied body establishes **three explicit reads in total** after assignment: one for `0x21` and two for computed byte stores. It does not establish a third later computed store sourced from `0x67`. Other later byte stores use saved values or truncated computed addresses.

### Remaining mutations and final transfer

One linked-base byte store uses a signed right shift of a loaded word. The shift count is the low eight bits of the field base and can exceed 31. Signed-shift convention and oversized-count behavior remain **documented_only**.

Late effects also include:

- Replacing the field-base word at offset `0x10` with a captured word from offset `0x40`.
- Stores through publication-base offsets `0x44`, `0x54`, and `0x74`.
- Restoring linked-base offset `0x54` from its earlier saved offset-`0x14` word.
- Capturing field-base offset `0x14` before further byte and word mutations.
- Reading a terminal base from linked-base offset `0x04`.
- Storing halfword `0x65` through twice that terminal base.
- Reading the final transfer’s second argument from terminal-base offset `0x64`.
- Writing decimal 100 at offset `0x54` of another computed base.
- Reading the final transfer’s fourth argument from terminal-base offset `0x74`.
- Performing further computed writes, ending with a word store through twice the saved second argument.

The final transfer uses captured values. Substituting later rereads is not justified.

## Side effects

Direct effects include 8-, 16-, and 32-bit memory accesses, one fixed-storage read of unresolved width, and an unresolved indirect transfer. Dependencies may add arbitrary effects.

Meaningful fixed-storage order includes the unresolved-width `0x05` read, byte write at `0x01`, later word write at `0x54`, byte write at `0x15`, publication-base load, word write at `0x60`, repeated publication, word capture at `0xA9`, byte write at `0x3A`, and the later `0x67` assignment/read sequence involving `0x21`.

Computed accesses interleave with these operations and may alias them. This ordering does not imply disjoint fixed objects.

Ownership is **documented_only**. No allocator, deallocator, ownership transfer, lifetime guarantee, or global-preservation promise is established. No rollback, synchronization, atomicity, or reentrancy guarantee is visible.

There are no explicit bounds checks, null checks, recoverable error branches, or failure statuses. Faults and nonreturning dependencies may leave partial mutations.

## Dependencies

F-6df2e1cea3ea is the only supplied opaque function ID. No opaque IDs resolve its indirect dependencies; none are invented here.

| Dependency | Established requirement | Unresolved property |
|---|---|---|
| Initial indirect target | Incoming control-derived address | Identity and interface |
| Subsequent targets | Ordered calls through constructed addresses | Effects, calling conventions, termination |
| Paired and secondary results | Consumed words and field extractions | Semantic meaning and producer guarantees |
| Fixed and computed storage | Ordered accesses at represented widths | Accessibility, aliasing, lifetime, byte layout |
| Final target | Four displayed arguments | Identity, effects, return convention |

No dependency is assumed to preserve globals or retain, transfer, or release referenced memory.

## Acceptance examples

1. **Argument extraction.** For third argument `0xAB123456`, its eight-bit contribution is `0xAB`; its separate nine-bit contribution is `0x156`. This does not prove whole-execution equivalence for inputs differing in other bits.

2. **Truncation.** Under 32-bit bit-vector semantics, extracting bits 4–23 of `0x00FFFFFF` yields `0xFFFFF`; shifting left 13 yields `0xFFFFE000`, not a widened result.

3. **Compact boundary.** When the relevant result’s low seven bits are all set, the compact quantity is 508 and its doubled base is 1016. Valid execution still requires accessible storage.

4. **Unknown fixed-read width.** An implementation that always reads exactly one byte at fixed address `0x05` is not justified by this evidence alone. The explicit computed-base byte reads at offset `0x05` do require byte-width acquisition.

5. **Publication capture before fixed overwrite.** If the publication-base storage overlaps `0x60`, the base used for publication must reflect its earlier load, not a reload after the `0x20` store.

6. **Late capture ordering.** If fixed byte `0x3A` overlaps the subsequently saved linked-base offset-`0x14` word, that word capture occurs after the byte write. The earlier `0xA9` capture remains saved.

7. **Changing `0x67`.** If an intervening computed store changes byte `0x67` between its two later computed-store reads, their destination bytes may differ. Reusing the initial assignment is not acceptable.

8. **Failure.** An unavailable indirect target provides no established failure return or rollback. Earlier effects may remain.

## Uncertainties

The following remain **documented_only**:

- Application purpose and complete calling convention.
- Recovered control-flow fidelity and dependency identities.
- Secondary-result production and dependency effects.
- Access width and conversion details of the symbolic fixed read associated with `0x05`.
- Target endianness, alignment, address wrapping, and exceptional arithmetic.
- Signed right-shift and oversized shift-count behavior.
- Storage accessibility, ownership, lifetime, and alias relationships.
- Numerical multiplicity of the repeated publication region in this contract.
- Normal termination and dependency error behavior.

The feedback’s additional computed read of `0x67` is not supported by the supplied body; it is not added as an assumed effect.

## Evidence

This contract uses static examination of the supplied interface, argument transformations, indirect-call sites, access widths, address derivations, repeated stores, fixed-storage ordering, saved-value dependencies, and final transfer.

The supplied body distinguishes the unresolved-width symbolic fixed read associated with `0x05` from explicit computed byte reads. It places the publication-base load before the `0x60` store, and the `0x3A` byte write between specified late captures. It explicitly represents one read of `0x67` for `0x21` and two later reads for computed stores.

Arithmetic deductions are marked **[INFERENCE]** where dependent on 32-bit bit-vector semantics. No implementation was executed, no runtime tests were performed, and no external dependency contracts were available.
