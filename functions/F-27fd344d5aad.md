<!-- Trusted provenance: F-27fd344d5aad; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-27fd344d5aad

## Purpose

F-27fd344d5aad initializes selected fields in an object-like memory region, performs seven lookup-and-conversion sequences, publishes four converted values through globally selected destinations, applies up to four byte-sized tags, and invokes a final setup sequence.

Its behavior depends on mutable global state read at specific points during execution. The metadata value, lookup tokens, and publication destinations are not established constants. Moving these reads to function entry could change behavior when dependencies modify globals.

The initial storage address and returned address are derived separately. Their equality is not established locally.

“Object,” “manager,” “resource,” and “tag” describe observable roles. Their application-level meanings, ownership rules, and dependency internals remain unknown.

## Interface

### Arguments and result

The function accepts three 32-bit values:

| Argument | Established use |
|---|---|
| First | Passed unchanged to F-77fb38f1315a, followed by a 32-bit zero. |
| Second | Written as 32 bits at initial-storage offset `0x2C`. |
| Third | Written as 32 bits at initial-storage offset `0x28`. |

There is no local argument validation. The first argument is not established to be an address, size, handle, or allocation request.

The result is a 32-bit address-like value. The following notation distinguishes the relevant addresses:

- **O**: result of F-77fb38f1315a.
- **R**: result of F-b2ce16bfaeab when called with `O + 0x7C`.
- **B**: `R − 0x7C`, returned on normal completion.

Early initialization uses O. Later object-field accesses and final setup use B or equivalent offsets from R. No local comparison requires B to equal O.

### Widths and global-state notation

All offsets are byte offsets. A word is 32 bits; a byte is 8 bits. Address-like values and publication-slot contents are represented as 32 bits.

The contract names global roles without exposing their source identifiers:

| Name | Role |
|---|---|
| **M** | Global supplying the initial 32-bit metadata value. |
| **T0–T6** | Seven globals supplying lookup-token values. |
| **H0–H3** | Four globals containing publication-destination addresses. |
| **C0–C2** | Addresses captured from H0–H2 for later slot-content reads. |
| **V0–V6** | Results of the seven conversion calls. |

H0–H3 are address-holder globals, not the publication slots themselves. A read of an address holder obtains an address; a subsequent store through that address changes the selected slot.

None of M, T0–T6, or H0–H3 is established to be immutable.

### Operational preconditions

Normal completion requires valid storage for every reached direct access and valid call targets for every reached invocation. In particular:

- O must support the early field writes and helper calls.
- R must yield valid later addresses after the specified offset adjustments.
- Each address sampled from H0–H3 must support its associated publication store.
- C0–C2 must support their later slot-content reads.
- Each nonzero tag target must support a byte write at offset `0xD0`.
- Each indirectly used manager must support a dispatch-table read and invocation through the entry at byte offset `0xF0`.

These are requirements on a successful execution, not checks performed by the function.

The representation uses 32-bit address arithmetic. It supplies no local overflow check, portable out-of-range policy, alignment repair, or invalid-address recovery. Endianness and platform fault behavior are not established.

## Behavior

### Initial storage and metadata sampling

First, F-77fb38f1315a receives the first argument and zero. After it returns O, the function reads M and writes the sampled 32-bit value at `O + 0x00`.

This timing is observable: F-77fb38f1315a may change M before the read. Substituting a constant or taking an entry-time snapshot is not equivalent without an additional stability guarantee.

The remaining early operations occur in this order:

| Order | Target | Operation |
|---|---|---|
| 1 | `O + 0x2C` | Write the second argument as 32 bits. |
| 2 | `O + 0x28` | Write the third argument as 32 bits. |
| 3 | `O + 0x30` | Invoke F-U1 with this address and `0x2C`. |
| 4 | `O + 0x5C` | Write the 32-bit pattern `0xFFFFFFFF`. |
| 5 | `O + 0x60` | Write a 32-bit zero. |
| 6 | `O + 0x64` | Write a 32-bit zero. |
| 7 | `O + 0x68` | Write one zero byte. |
| 8 | `O + 0x6C` | Write a 32-bit zero. |
| 9 | `O + 0x70` | Write one byte containing one. |

F-U1 is a document-local opaque label for the unmapped external helper. Its second argument is numerically 44. The source does not establish that this argument is a byte count or that the helper clears memory.

The byte stores at `0x68` and `0x70` do not initialize the following three bytes. Unlisted bytes have no direct initialization guarantee from this sequence.

Next, F-b2ce16bfaeab receives `O + 0x7C`. Its result establishes R and the adjusted base B.

### Lookup-token sampling and conversion

Each of the seven lookup stages begins with a separate call to F-938c950e5046. After that call returns, the corresponding token global is read for the lookup invocation.

Consequently, every token read occurs:

1. After that stage’s manager-acquisition call returns.
2. Before that stage’s direct or indirect lookup invocation.
3. Without an established requirement that its value match an earlier or entry-time value.

An earlier dependency, including the immediately preceding manager-acquisition call, may change the next token value.

| Stage | Token global | Lookup path | Conversion selector | Destination |
|---|---|---|---:|---|
| 0 | T0 | F-9e50c66a96dd | `0x8B00` | `B + 0x14` |
| 1 | T1 | F-U2 | `0x8B00` | Destination selected through H0 |
| 2 | T2 | F-U2 | `0x8B00` | Destination selected through H1 |
| 3 | T3 | F-U2 | `0x8B00` | Destination selected through H2 |
| 4 | T4 | F-U2 | `0x8B00` | Destination selected through H3 |
| 5 | T5 | F-9e50c66a96dd | `0x8E00` | `B + 0x1C` |
| 6 | T6 | F-U2 | `0x8E00` | `B + 0x20` |

For the direct path, F-9e50c66a96dd receives the freshly obtained manager and sampled token.

For the indirect path, F-U2 denotes the operation selected by reading the manager’s dispatch-table pointer and selecting the entry at byte offset `0xF0`. It receives the manager and sampled token. There is no local null-manager check. The exact evaluation order among the token read and the dispatch-address reads within that invocation expression is not established.

Every lookup result is passed to F-682adfd5a561 with the selector shown above. Its returned 32-bit value becomes the corresponding V value.

Stage 0 writes V0 at `B + 0x14` before stage 1 begins. Stages 1–4 perform the publications described next. Tagging follows stage 4 and precedes stage 5. Stages 5 and 6 write V5 and V6 at their listed object offsets.

### Publication-destination sampling

Publication destinations are acquired after their corresponding conversions return, not before lookup or at function entry.

For each of stages 1, 2, and 3, the representation shows two separate uses of the associated address-holder global, in this order:

1. Read the address-holder global and retain the resulting address as C0, C1, or C2.
2. Read that address-holder global again as the destination address for the publication store.
3. Store the corresponding converted value as 32 bits through that publication address.

Thus, stage 1 captures C0 from H0 before publishing V1 through the address then supplied by H0. Stages 2 and 3 do the same for H1/V2 and H2/V3.

The source does not collapse the retained-address capture and publication-destination read into one read. With stable address-holder contents, both reads select the same address. If those contents change between reads under an applicable execution model, the captured address and publication destination need not match. No synchronization guarantee is supplied.

For stage 4, after V4 is returned, the function reads H3 and writes V4 through that address. It retains V4 for tagging but does not retain a slot address from H3.

The complete publication order is V1, V2, V3, then V4. Dependencies in intervening stages may change address-holder globals before subsequent publications acquire their destinations. Reusing an entry-time snapshot of H0–H3 would therefore change the contract.

Later tagging reads use C0–C2. They do not reread H0–H2 to rediscover the slots.

### Slot-content reads and tagging

After all four publications, the function performs the following accesses in order:

1. Read one 32-bit value through C0.
2. Write that sampled value at `B + 0x18`.
3. If the sampled value is nonzero, write a zero byte at that value plus `0xD0`.
4. Read the 32-bit contents through C1 for a nonzero test. If nonzero, read through C1 again and use that second value as the address for a byte write of one at offset `0xD0`.
5. Read the 32-bit contents through C2 for a nonzero test. If nonzero, read through C2 again and use that second value as the address for a byte write of two at offset `0xD0`.
6. If retained V4 is nonzero, write a byte containing three at `V4 + 0xD0`.

There are three distinct mechanisms:

- The zero tag uses one retained slot address and one sampled slot value.
- Tags one and two each use a retained slot address but potentially two slot-content reads.
- Tag three uses the retained conversion result, not a later slot read.

For tags one and two, a successful first nonzero test does not locally validate the second sampled value. If slot contents change between reads, the eventual target follows the second value, even if that value is zero. There is no second null check.

Null values suppress only the corresponding conditional tag operation. They do not terminate initialization or suppress the remaining lookups and final setup.

### Final setup and branch

After V5 and V6 have been stored, the following calls occur in order:

| Order | Dependency | Explicit arguments |
|---|---|---|
| 1 | F-465a3937497e | B, one |
| 2 | F-861a66c91b48 | B, `B + 0x30`, zero, zero |
| 3 | F-b00260d1087a | `B + 0x74` |
| 4 | F-338d16f9ce72 | `B + 0x74` |
| 5 | F-7e4f952e879e | None shown |
| 6 | F-69d68a9d62e5 | None shown |

The result of F-69d68a9d62e5 is compared with exactly one:

- Exactly one selects F-abdb265693e5, with no explicit arguments.
- Every other result selects F-c826609a57a6 with B.

After the selected branch returns normally, F-1333331b2da1 receives `B + 0x74`. The function then returns B.

The predicate is not a generic nonzero test. Zero, two, and the 32-bit all-ones pattern select F-c826609a57a6.

## Side effects

### Direct writes and their limits

Reached stages directly write:

- The listed initial fields relative to O.
- Four 32-bit resource-related fields at `B + 0x14`, `B + 0x18`, `B + 0x1C`, and `B + 0x20`.
- Four 32-bit publication destinations obtained through H0–H3.
- Up to four tag bytes at resource-relative offset `0xD0`.

Zero conversion results are still stored. No local resource-failure return is generated.

These are ordered write events, not unconditional final-state guarantees. Later dependencies may change the same storage, globals, or pointed-to values.

### Aliasing

No distinctness guarantee is established for O, B, publication destinations, captured slot addresses, resource targets, or other accessed storage.

Aliasing may change later reads. For example, overlapping publication destinations may cause later publications to replace values subsequently read through C0–C2. A tag write may also affect storage used by a later access if those locations overlap.

When all four tag targets are the same valid address and no intervening mutation changes the targets, the byte receives zero, one, two, and three in order. Three is its value immediately after tagging, not necessarily after later dependencies return.

### Ownership, failure, and synchronization

No ownership transfer, allocation responsibility, reference-count operation, or release obligation is established for O, B, managers, lookup results, converted values, or overwritten publication contents.

There is no explicit local release of an overwritten slot value and no visible rollback. A fault or nonlocal exit may leave earlier writes observable. The final call on `B + 0x74` is guaranteed only along the shown normally returning path.

No local synchronization mechanism is shown. The source does not establish thread safety, atomicity, race behavior, or reentrancy.

## Dependencies

| Opaque ID | Established invocation role |
|---|---|
| F-77fb38f1315a | Receives the first argument and zero; returns O before M is read. |
| F-U1 | Unmapped external helper receiving `O + 0x30` and `0x2C`. |
| F-b2ce16bfaeab | Receives `O + 0x7C`; returns R. |
| F-938c950e5046 | Called separately before each of seven token reads and lookups. |
| F-9e50c66a96dd | Direct lookup operation for stages 0 and 5. |
| F-U2 | Indirect lookup selected through dispatch-table offset `0xF0`. |
| F-682adfd5a561 | Converts each lookup result using the specified selector. |
| F-465a3937497e | Receives B and one. |
| F-861a66c91b48 | Receives B, `B + 0x30`, zero, zero. |
| F-b00260d1087a | First final-stage call on `B + 0x74`. |
| F-338d16f9ce72 | Second final-stage call on `B + 0x74`. |
| F-7e4f952e879e | Argument-free call immediately before the branch query. |
| F-69d68a9d62e5 | Supplies the result compared with one. |
| F-abdb265693e5 | Selected for exactly one. |
| F-c826609a57a6 | Receives B for every other query result. |
| F-1333331b2da1 | Final call on `B + 0x74` before normal return. |

F-U1 and F-U2 are document-local labels, not recovered identities. F-U2 may resolve to different targets for different managers. No dependency is assumed to preserve globals or object fields, retain resources, or release memory.

## Acceptance examples

These are conditional static acceptance scenarios, not executed tests. All reached accesses and calls must be valid.

### Metadata changed by the first dependency

If M initially contains A and F-77fb38f1315a changes it to C before returning, with no intervening change before sampling, the metadata write stores C at `O + 0x00`. An entry-time snapshot storing A would not conform.

### Token changed during manager acquisition

If a stage’s token global initially contains A but that stage’s F-938c950e5046 call changes it to C, the following lookup receives C, assuming no further change before sampling. This applies independently to all seven stages.

### Publication destination changed by conversion

If H1 points to slot P before the stage-2 conversion, but that conversion changes H1 to Q and it remains stable through capture and publication, C1 becomes Q and V2 is written to Q. The earlier P is not used merely because it was present before conversion.

### Retained address survives address-holder replacement

If stage 1 captures C0 as P and publishes V1 to P, but a later dependency changes H0 to Q, tagging still reads through P. The value sampled from P is copied to `B + 0x18` and, if nonzero, receives tag zero. H0 is not reread for this purpose.

### Capture and publication are separate accesses

Conditionally, if the execution environment permits H0 to change from P to Q between its capture read and publication-address read, C0 remains P while the publication writes V1 to Q. Later tagging reads P. An implementation that unconditionally merges the two reads would omit this distinction.

This example does not establish that concurrent mutation is supported or well-defined on the target platform.

### Retained fourth conversion result

If V4 is valid nonzero address D, publication writes D to its selected slot. A later change to that slot or H3 does not redirect tag three: it still targets `D + 0xD0`.

### Zero resources

If all conversions return zero, each publication selects valid storage, the captured slots contain zero when tested, and no other effect changes those observations:

- The conversion-result fields receive zero when assigned.
- The captured C0 value copied to `B + 0x18` is zero.
- No tag write occurs.
- Final setup and branch selection still execute.

### Distinct initial and returned storage

If B differs from O, early field writes still target O. Resource-field writes and final object-relative calls target B. Normal return yields B.

### Exact branch selection

A query result of one selects only F-abdb265693e5. A result of zero, two, or `0xFFFFFFFF` selects only F-c826609a57a6 with B. After either selected call returns normally, F-1333331b2da1 executes before B is returned.

## Uncertainties

The following remain **documented_only**:

- Whether O is allocated, constructed, or reused.
- Whether B normally equals O.
- The semantic meanings and allowed values of metadata, tokens, selectors, and tags.
- The identities and internals of F-U1 and indirect F-U2 targets.
- F-U1’s affected memory extent and interpretation of `0x2C`.
- Which globals or storage locations any dependency may modify.
- Ownership, borrowing, retention, release, and lifetime rules.
- Whether the calls on `B + 0x74` implement locking, a guard, or another mechanism.
- Target-platform concurrency, atomicity, alignment, overflow, and fault semantics.
- Cleanup behavior after exceptions or other nonlocal exits.
- Calling-convention details beyond the explicit arguments shown.

Constructor-like initialization, lookup, conversion, and ordinal tagging are inferred intent. They do not replace the specified access widths, sampling points, and ordering.

## Evidence

Established static evidence consists of the supplied function representation and dependency mapping: explicit arguments, access widths, offsets, separate global reads, capture-before-publication ordering, retained slot addresses, repeated slot-content reads, the exact comparison with one, and the adjusted return value.

The supplied representation supports mutable-global sampling semantics; it does not establish immutable metadata, fixed lookup-token values, or entry-time publication destinations.

Dependency internals and application semantics are not established by the supplied material. No runtime tests, traces, allocation observations, or concurrency experiments have been performed. Acceptance examples express conditional obligations, not observed executions.
