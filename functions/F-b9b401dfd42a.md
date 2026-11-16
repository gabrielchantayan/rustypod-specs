<!-- Trusted provenance: F-b9b401dfd42a; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-b9b401dfd42a

## Purpose

F-b9b401dfd42a performs an ordered sequence of indirect invocations and integer transformations, followed by direct memory mutations and a final indirect transfer.

**Established static evidence:** the supplied representation declares no explicit parameters or scalar return value. Observable effects include dependency invocations, 8-, 16-, and 32-bit memory accesses, repeated writes, pointer-like traversal, and accesses to fixed low addresses. No validation branch, recoverable error result, or rollback is visible.

**Inferred intent:** some accesses resemble linked-record updates. Some constants coincide with printable characters. Neither observation establishes a container operation, text formatter, initializer, or hardware protocol.

This contract describes supported static behavior, not an unconditional executable replacement. Entry conventions, dependency contracts, memory mappings, and target-specific arithmetic remain partly unresolved. Unresolved behavior is marked `documented_only`.

## Interface

### Inputs

Although the represented interface has no explicit arguments, execution consumes:

- Two incoming 32-bit machine-state values used in early left shifts.
- Incoming continuation-related state used to derive the first indirect destination.
- Primary and secondary results of indirect invocations.
- Existing bytes and words at computed and fixed addresses.
- A late-loaded 32-bit stored address base.
- An unresolved destination for the final indirect transfer.

“No explicit parameters” therefore does not establish independence from caller state.

Exact register assignments, preservation obligations, stack layout, and valid entry conditions are `documented_only`. The representation does not establish a conventional parameterless application API.

### Outputs and completion

There is no explicit scalar result. Outputs consist of memory changes, dependency effects, and control transfer.

The final indirect invocation has four visible argument positions:

| Position | Established value |
|---|---|
| First | `0x72` |
| Second | A 32-bit value obtained by late traversal through a word at offset `0x64` |
| Third | A retained 32-bit word read earlier from computed offset `0x14` |
| Fourth | A later 32-bit word read from another computed base at offset `0x74` |

A return is represented after this invocation. Whether the invocation returns normally, dispatches elsewhere, or requires another calling convention is `documented_only`.

### Widths and layout

| Access or value | Width | Interpretation |
|---|---:|---|
| Byte | 8 bits | Byte stores retain only the low eight bits |
| Halfword | 16 bits | Explicit stored constants include `0x003a` and `0x0065` |
| Word | 32 bits | Used as data, scaled values, and address bases |
| Paired invocation result | 64 bits | Low and high 32-bit portions are extracted separately |

Several byte reads are explicitly zero-extended for word storage. A byte value of `0xff` then becomes `0x000000ff`, not a sign-extended word.

Offsets describe accesses, not a proven object schema. Byte offsets include 1, 5, 8, 9, `0x0d`, `0x11`, `0x15`, and `0x19`. Word offsets include 4, `0x10`, `0x14`, `0x20`, `0x24`, `0x34`, `0x40`, `0x44`, `0x54`, `0x64`, and `0x74`.

Byte order, alignment support, and record boundaries are `documented_only`. A halfword containing `0x0065` does not establish a particular two-byte string.

### Preconditions and ownership

Completion requires every reached invocation destination and memory access to be valid in the execution environment. No visible null, bounds, alignment, or permission checks enforce this requirement.

No allocation, release, reference-count adjustment, or ownership transfer is identifiable among the direct operations. Unknown dependencies may perform such actions; their contracts are unavailable. Integer-derived addresses must not be treated as owned objects.

## Behavior

### Indirect invocation phase

A long straight-line invocation phase precedes the explicit memory-update region.

The first destination is derived from incoming continuation-related state plus `0xffe`. Later destinations are largely derived from embedded continuation-like values with the same displacement. The representation shows computed invocation rather than an explicit function-pointer-table lookup.

Invocations occur in represented order. Visible argument counts range from zero through four. A site with no explicit arguments may still consume implicit machine state.

Result transformations include:

- Left shifts of incoming values by 7 and 12 bits.
- Frequent left shifts of primary results by 8 bits.
- Forwarding a high result word alongside a shifted low word.
- Repeating a transformed value in several argument positions.
- Interleaving explicit zeros with result-derived values.
- Masking low portions of words before further shifts.
- Logical right shifts of unsigned values.
- Extracting upper portions of paired results.

The visible sequence does not test status values before continuing. Zero is transformed or forwarded at applicable sites, not visibly interpreted as failure.

The complete invocation-by-invocation argument trace is `documented_only` in this contract. This is a limitation of the documented coverage, not evidence that the source lacks an ordered trace. Invocation order, repeated arguments, and apparently redundant calls must not be replaced with assumptions of purity or equivalence.

### Arithmetic boundaries

Most intermediates are represented as 32-bit quantities. Width-sensitive examples below use a **32-bit bit-vector model**: arithmetic results are reduced modulo \(2^{32}\), and unsigned right shifts insert zeros. This model is conditional, not proof of all target instruction semantics.

Under that model:

- A left shift by 8 discards the original high byte.
- Multiplication by `0x02000000` retains contributions from only seven original low bits.
- Successive left shifts by 28 and 28 produce zero.
- Successive left shifts by 31 and 30 produce zero.
- Explicitly zeroed intermediates remain zero through the visible shifts.
- An unsigned 64-bit result shifted right by 34, 36, or 44 corresponds to its high word shifted logically right by 2, 4, or 12.

These simplifications do not authorize removing intervening invocations.

A late right shift has a signed 32-bit operand and a count taken from the low byte of an address-derived value. Counts may exceed 31. Large-count behavior, negative-operand behavior, signed overflow, and address wraparound are `documented_only`; host-language rules must not be substituted silently.

### Initial direct mutations

The final paired result of the invocation phase supplies low and high words used immediately in addressing.

The opening effects preserve this order: a word is written at high-result-plus-`0x14`; a high-result-derived byte is written at another computed base plus 9; a retained shifted value is written at low-result-times-256-plus-`0x54`; and the byte at the earlier plus-9 destination is overwritten with another retained low byte.

A word at a shifted base plus `0x14` is cleared before the byte at that base plus 5 is read. That byte is zero-extended for a later word store at offset `0x34`.

Another base is obtained from a result shifted left by 8, masked with `0x7fff`, shifted logically right by 6, and doubled. Under the bit-vector model, the resulting base ranges from zero through `0x3f8`, in multiples of eight. This numerical range establishes no valid memory mapping.

At that base, word clears at offset `0x14` precede two identical byte stores at offset `0x15`. These accesses overlap. Their widths and repeated occurrence remain part of the represented behavior.

After those byte stores, the routine reads a byte from fixed address `0x05`, writes a byte to fixed address `0x01`, and uses the zero-extended read byte as another address base. Subsequent computed writes precede an old-word read at offset `0x54`, replacement of that word, and later use of the retained old word as a base.

### Traversal and overlapping updates

The next region repeatedly loads and replaces words, follows words at offsets `0x64` and `0x74`, writes bases into their own offset fields, and distributes low bytes between computed destinations.

Notable dependencies include:

- A word loaded before replacement at offset `0x54` remains available for later addressing.
- A byte read at one base plus 5 becomes an address base for a word read at offset `0x74`.
- Two reads from the same offset `0x74` participate in one reached-location update. They are not established as one cached read.
- Repeated byte reads at offset 5 occur after intervening mutations.
- Fixed word address `0x54` is written between computed updates.
- Fixed byte address `0x15` is written after two byte writes at a computed offset `0x15`.

No distinctness checks separate these bases or distinguish computed destinations from fixed-address storage. Later reads therefore use memory after preceding writes. An entry-time snapshot of pointer-source words is unsupported.

### Constant stores and retained-base publication

A later region writes numeric `0x3a` as a byte at four times a reached base, as a halfword at eight times that base, and as a word at the four-times base plus `0x20`.

At another computed destination plus `0x20`, the ordered word values are:

1. `0x20`.
2. `0x2f`.
3. A zero-extended byte freshly read from that base plus 4.
4. `0x20`.

The byte read belongs between the second and third word stores. Aliasing with the earlier stores is not excluded.

The routine next reads a dedicated stored 32-bit address base. **That read precedes the fixed word write of `0x20` at address `0x60`.** The retained result, not a reload after the `0x60` write, supplies the following publication-like destinations. Earlier computed writes may already have affected the stored slot.

A long repeated region then writes a scaled reached value at offset `0x54`, a byte at retained-base-plus-`0x19`, a byte at another retained base plus `0x11`, and a word at retained-base-plus-`0x34`.

The opening group differs from the following groups in its scaled destination and published word value. The region does not visibly advance through array elements. Its exact repetition count is `documented_only` here; neither one effective update nor an invented count is acceptable.

### Final-region ordering anchors

For this section, **publication base** means the retained stored address base; **working base** means the other retained base used for the repeated plus-`0x11` byte writes; **traversal base** means the retained doubled reached value; and **expanded base** means the retained four-times reached value. These are descriptive roles, not object types or claims of distinct storage.

The final region starts with an established ordering boundary:

- The word at fixed address `0xa9` is read and retained.
- A word at expanded-base-plus-`0x74` is read and retained.
- A byte is written at fixed address `0x3a`.
- Only afterward are words retained from traversal-base-plus-`0x14` and working-base-plus-4, in that order.

The `0x3a` byte is derived from eight times a retained adjusted value plus `0x30`, narrowed to eight bits. Its write can overlap either later word source. Those later words must not be treated as values captured before the fixed byte write.

The following computed mutations distribute retained low bytes to offsets `0x0d` and 9, store byte `0x6c` at working-base-plus-8, and store the traversal base into its own offset `0x54`. Only then is the low byte of the retained `0xa9` word written to fixed address `0x67`.

### Three distinct reads of address `0x67`

The fixed byte is read three times after that write. Each read has a separate position among computed accesses.

**First read:** a word is read from traversal-base-plus-`0x44` after the `0x67` write and before the first `0x67` read. The first byte read from `0x67` supplies the byte write to fixed address `0x21`.

Between this first read and the second, the routine:

- Distributes the newly read traversal word through byte and word stores.
- Writes the variable-count signed-shift result at traversal-base-plus-1.
- Replaces publication-base-plus-`0x54` with the working base.
- Reads a word at working-base-plus-`0x40`.
- Writes the retained working-base-plus-4 word at publication-base-plus-`0x74`.
- Writes the newly read plus-`0x40` word at working-base-plus-`0x10`.
- Writes the traversal base at publication-base-plus-`0x44`.

The byte store at working-base-plus-`0x10` precedes the word replacement at that same offset. The second fresh read from `0x67` then supplies a byte store at working-base-plus-9.

**Third read:** between the second and third reads, a byte is stored at traversal-base-plus-9, the retained `0xa9` word is stored at working-base-plus-`0x44`, and a word is read from working-base-plus-`0x54` for subsequent traversal. The third fresh read from `0x67` supplies another byte store at working-base-plus-9.

Computed stores in these intervals may overlap `0x67`. Even the second read’s own destination may overlap the later read source. Thus neither later byte read is required to return the byte originally assigned from `0xa9`, nor are the three read values required to agree.

### Remaining traversal and transfer

After the third `0x67` read and its byte store, the reached base receives a low-byte update at offset `0x0d`. The traversal base’s offset `0x54` is restored from its earlier retained plus-`0x14` word.

Further interleaved operations read working-base-plus-`0x14`, write low bytes at offsets `0x11` and 9, replace working-base-plus-`0x44`, and read traversal-base-plus-4. The last read supplies an address whose doubled value receives halfword `0x0065`.

Late traversal follows words at offsets `0x64` and `0x74`. A reached location receives word 100 at offset `0x54`; another reached location receives the value later used as the second final argument; and a reached base is stored at twice that argument value.

The final invocation receives the values described in Interface. Aliasing may affect late traversal and argument values. No normalized final object state is established.

## Side effects

### Fixed-address inventory

This table supplements, but does not replace, the ordering requirements above.

| Address | Explicit access |
|---|---|
| `0x01` | Byte write |
| `0x05` | Byte read |
| `0x15` | Byte write |
| `0x21` | Byte write supplied by the first final-region read of `0x67` |
| `0x3a` | Byte write before the final-region retained reads at computed offsets `0x14` and 4 |
| `0x54` | Word write |
| `0x60` | Word write of `0x20`, after the stored address-base read |
| `0x67` | Byte write, followed by three distinct byte reads |
| `0xa9` | Word read before the final-region computed plus-`0x74` read |

Computed accesses can introduce additional accesses to these same locations. The table lists explicit fixed-address effects, not exclusive access counts or independent global objects.

### Failure and concurrency

No recovery, diagnostic, rollback, or failure-atomicity guarantee is visible. A faulting access or nonreturning dependency may leave earlier effects completed. Fault delivery is environment-dependent and `documented_only`.

No visible locking, atomic operation, or synchronization establishes thread safety or reentrancy. Repeated accesses do not by themselves establish device-memory semantics, but must remain distinct in any trace-sensitive interpretation.

Unknown dependencies must not be assumed to preserve globals, memory, or machine state beyond a separately established calling convention.

## Dependencies

No canonical callee IDs are supplied. The following opaque labels identify unresolved roles only:

| Opaque ID | Role | Unresolved contract |
|---|---|---|
| D-001 | First indirect invocation | Identity, ABI, effects, return behavior |
| D-002 | Remaining ordered invocations before direct mutation | Individual identities, arguments, effects, preservation |
| D-003 | Final four-position indirect invocation | Destination, ABI, effects, return behavior |

D-002 denotes a collection of sites, not one known function. These labels assert neither equivalence nor distinctness of targets.

No dependency is established as an allocator, releaser, pure arithmetic helper, or memory-preserving operation.

## Acceptance examples

These are conditional static criteria, not executed tests.

### Width and widening

Under the bit-vector model, shifting `0x12345678` left by 8 yields `0x34567800`. Storing that original value as a byte writes only `0x78`. A zero-extended byte read of `0xff` supplies word `0x000000ff`.

### Overlapping mixed-width updates

Assuming writable ordinary memory, a word clear at base-plus-`0x14` followed by two byte writes at base-plus-`0x15` retains both byte-write events in a trace. Matching final contents alone does not establish the required repeated accesses.

### Fixed write before retained word reads

If address `0x3a` lies within either later word source at traversal-base-plus-`0x14` or working-base-plus-4, that word read observes the preceding byte write according to target byte order and memory semantics. Capturing both words before the write is unacceptable.

### Stored address-base ordering

A trace of the publication boundary must show the stored 32-bit address-base read before the word write of `0x20` to `0x60`. Subsequent publication destinations derive from the retained read result. Reloading the slot after the fixed write does not establish equivalent behavior.

### Distinct `0x67` observations

If intervening computed stores change `0x67`, the byte copied to `0x21` and the two later bytes stored at working-base-plus-9 may differ. Acceptance must preserve three fresh reads and their intervening accesses, rather than forwarding the original low byte of the retained `0xa9` word.

### Intermediate overwrites

The four word stores at the identified plus-`0x20` destination expose `0x20`, `0x2f`, the freshly widened byte, and `0x20`, in order. A final-state-only observation cannot verify the intermediate effects.

### Zero results and invalid addresses

A zero result used by an eight-bit left shift produces zero under the bit-vector model; no visible short circuit follows. Completion still depends on later calls and accesses being valid.

If a reached address is unmapped, no graceful failure result is established. Earlier effects and actual fault delivery remain environment-dependent.

## Uncertainties

The following remain `documented_only`:

- Application purpose and success condition.
- Entry ABI, implicit inputs, and preservation requirements.
- Dependency identities and contracts.
- Whether paired results reflect a true 64-bit return convention.
- Complete per-invocation argument trace and exact repeated-region count.
- Target signed arithmetic, large shift counts, and address wraparound.
- Endianness, alignment, permissions, and memory mappings.
- Record boundaries, ownership, alias restrictions, and device semantics.
- Fault handling, concurrency guarantees, and final transfer behavior.

The documented ordering of the `0xa9` read, computed plus-`0x74` read, `0x3a` write, later retained word reads, stored address-base read before `0x60`, and three separate `0x67` reads is established static evidence, not an unresolved ordering question.

## Evidence

This contract uses static examination of the supplied source representation. Prior prose, comments, and feedback do not independently establish behavior.

Supported observations include invocation order, result splitting, widths, masks, shifts, constants, offsets, repeated accesses, fixed-address reads and writes, the stated ordering boundaries, and the final indirect invocation.

“Record-like” and “publication” are descriptive labels, not established application intent. Numerical simplifications are conditional on the stated bit-vector model.

No runtime tests, execution traces, dependency verification, target-instruction validation, or memory-map validation have been performed. No snapshot hash or independently verified provenance is asserted.
