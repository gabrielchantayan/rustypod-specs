<!-- Trusted provenance: F-1e4d0fba272a; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-1e4d0fba272a

## Purpose

F-1e4d0fba272a initializes a caller-supplied object, executes four status-gated dependency calls, and, if those calls succeed, publishes workspace results into the object. The successful gated path also inspects auxiliary metadata and optionally constructs indexed name mappings.

The operation is not transactional. Initial object writes precede the first status-producing call. Later publication and mapping can occur even when the function ultimately returns status `3`. There is no visible rollback.

**Established static evidence:** access widths, field offsets, direct-write ordering, dependency arguments, branch conditions, and status propagation.

**Inferred intent:** the diagnostic concerning missing character-string data and the distinguished string `.notdef` suggest font-related loading. Neither that interpretation nor conventional string-comparison semantics is required by this contract.

## Interface

### Argument and result

The function accepts one 32-bit address-like value identifying a mutable object and returns a 32-bit integer status.

No explicit validation checks the object address, object size, alignment, nested pointers, array capacities, or string storage. Every reached access requires suitable addressable storage. This contract does not assign recovery behavior to invalid accesses.

Four dependency calls gate further processing:

1. F-fb875b7a8097.
2. First call to F-973fbfc6bac0.
3. F-6a5c025e65c4.
4. Second call to F-973fbfc6bac0.

Each must return zero for the next stage to execute. The first nonzero result becomes the function’s returned status after final cleanup. If all four return zero, the status remains zero unless the later workspace check assigns `3`.

### Directly accessed object layout

All listed fields are 32 bits except the two explicitly identified bytes.

| Object offset | Direct use |
|---|---|
| `+0x64`, `+0x68` | Arguments to the first gated dependency |
| `+0xA8` | Receives `4` during initialization |
| `+0xAC` | One-byte flags field; bit 0 is cleared after gated success |
| `+0x110`, `+0x158` | Receive values from two static 32-bit locations |
| `+0x114`, `+0x118` | Receive `7` and `1`, respectively |
| `+0x16C` | One-byte mapping selector; only value `1` enables mapping |
| `+0x170` | Receives workspace `+0x60` after mapping |
| `+0x174`, `+0x178` | Receive signed 32-bit mapping bounds |
| `+0x17C` | Pointer to 16-bit mapping-index entries |
| `+0x180` | Pointer to 32-bit mapping-name pointers |
| `+0x184`, `+0x190`, `+0x194`, `+0x198` | Conditional publication destinations |
| `+0x188`, `+0x18C`, `+0x19C`, `+0x1A0`, `+0x1A4`, `+0x1A8` | Other publication destinations |
| `+0x1E4` | Read at entry and retained for dependency arguments |
| `+0x20C` | Pointer to auxiliary metadata |

Auxiliary metadata has a 32-bit first word at `+0`, a 32-bit word at `+4`, and one-byte record markers at `+0x58 + 12 × record index`. The word at `+4` serves first as a nonzero header test and later as an unsigned record count.

### Temporary storage

The function supplies a `0x174`-byte workspace and a separate 40-byte scratch buffer. Workspace offsets below are relative to that workspace, not to the object.

The workspace is passed to dependencies. Its complete initialization, contents, and resource ownership are not established. The scratch buffer is explicitly cleared only after the first gated call succeeds.

## Behavior

### 1. Initialize temporary state and object fields

The first direct object read obtains `+0x1E4`; this value is retained rather than reread for later dependency arguments.

F-U01 is then called with the workspace address and size `0x174`. Its behavior is unknown; the supplied evidence does not establish that it clears the workspace.

After that call, the function explicitly initializes workspace storage in this order:

1. Zero the 32-bit word at `+0x98`.
2. Zero the 32-bit word at `+0xA8`.
3. Zero the 32-bit word at `+0x148`.
4. Zero the 32-bit word at `+0x70`.
5. Zero the 32-bit word at `+0x110`.
6. Zero the byte at `+0x170`.
7. Zero the 32-bit word at `+0x60`.
8. Zero the 32-bit word at `+0xDC`.

It then writes object fields in this order:

1. `+0x114` receives `7`.
2. `+0x118` receives `1`.
3. `+0xA8` receives `4`.
4. `+0x158` receives a value read from one static 32-bit location.
5. `+0x110` receives a value read from another static 32-bit location.

The static values are not supplied as numeric constants.

F-fb875b7a8097 receives the workspace address, object `+0x68`, object `+0x64`, and the retained entry value from `+0x1E4`, in that argument order. The relative evaluation order of the two object-field reads is not established.

### 2. Execute the gated calls

A nonzero result from F-fb875b7a8097 skips directly to final cleanup.

Otherwise, all 40 scratch bytes are written to zero in ascending byte order. The following calls then execute, each only if its predecessor returned zero:

| Dependency | Arguments, in order |
|---|---|
| F-973fbfc6bac0 | Object, workspace, workspace `+0x4C`, workspace `+0x50`, scratch buffer |
| F-6a5c025e65c4 | Workspace, retained entry value from object `+0x1E4` |
| F-973fbfc6bac0 | Object, workspace, workspace `+0x54`, workspace `+0x58`, same scratch buffer |

The scratch buffer is not cleared between passes. The second pass can observe effects of the first pass and intervening dependency.

Any nonzero result skips the remaining gated calls and all subsequent direct metadata, publication, and mapping processing. Earlier writes and dependency effects are not undone.

### 3. Clear a flag and inspect metadata

After all four gated calls return zero, the function reads the byte at object `+0xAC`, clears bit 0, and writes the resulting byte back. This direct operation preserves the other seven bits.

It next reads object `+0x20C`.

- A null pointer skips the entire metadata phase without calling F-aa5254170a3e.
- For a nonnull pointer, the first metadata word is read.
- Only when that word is nonzero is metadata `+4` read for the initial header decision.
- If both words are nonzero, scanning is enabled.
- Otherwise, F-aa5254170a3e is called with the object. Object `+0x20C` is then reread; scanning is enabled only if that reread is nonnull.

The metadata returned after refresh is not subjected to another two-word header-validity test.

Scanning starts with unsigned 32-bit index zero. Every loop condition reloads the metadata pointer from object `+0x20C` and reads its current unsigned count at `+4`. For a visited record, the metadata pointer is read again to address the marker at `+0x58 + 12 × index`.

A nonzero marker advances the scan. A zero marker calls F-aa5254170a3e and immediately ends scanning. The scan is not restarted after that call. No return value from either refresh call is checked.

The repeated pointer reads are not null checks or synchronization guarantees.

### 4. Publish results and determine the post-gating status

The following actions occur in order:

1. Copy workspace `+0x98` to object `+0x19C`.
2. If workspace `+0x148` is nonzero:
   - Clear workspace `+0x148`.
   - Copy workspace `+0x138` to object `+0x190`.
   - Copy workspace `+0x13C` to object `+0x184`.
   - Copy workspace `+0x154` to object `+0x194`.
   - Copy workspace `+0x158` to object `+0x198`.
3. If workspace `+0xDC` is zero:
   - Call F-4457a9a44a68 with a fixed diagnostic indicating missing character-string data.
   - Assign status `3`.
4. Clear workspace `+0xDC`.
5. Perform these copies in the listed order:

| Object destination | Workspace source |
|---|---|
| `+0x188` | `+0xD0` |
| `+0x1A4` | `+0xE8` |
| `+0x1A8` | `+0xEC` |
| `+0x18C` | `+0x9C` |
| `+0x1A0` | `+0xB4` |

6. Clear workspace `+0x9C`, then workspace `+0xB4`.

The diagnostic call precedes the last five copies. Those copies use workspace values read after that call, not guaranteed pre-diagnostic snapshots.

Clearing workspace fields may indicate a resource-transfer convention, but that is inferred intent. The assignments alone do not establish ownership or release obligations.

### 5. Optionally construct mappings

Mapping executes only if the byte read from object `+0x16C` equals exactly `1`. Other byte values skip it.

The mapping phase maintains a signed 32-bit slot index, initially zero, and signed bound accumulators initially `32000` and `-32000`.

#### Per-slot accesses and defaults

Every outer-loop condition rereads the signed 32-bit workspace count at `+0x74`. A slot is visited only while its index is signed-less-than that current count. The count is not a fixed entry snapshot.

For each visited slot, the function performs these actions in order:

1. Read object `+0x17C` and write a 16-bit zero at that pointer plus twice the slot index.
2. Separately read object `+0x180` and write the 32-bit pointer to `.notdef` at that pointer plus four times the slot index.
3. Read the current input-array pointer from workspace `+0x7C`.
4. Fetch the slot’s 32-bit input-name pointer from that array.

Thus, the input-array pointer is reread for every visited slot, after the default output writes. A null fetched input pointer skips candidate processing for that slot.

For a nonnull input pointer, its pointer value is retained across candidate comparison calls. This does not establish that the pointed-to bytes remain unchanged.

#### Candidate search and selection

The candidate index starts at zero for each nonnull slot input. Every inner-loop condition rereads object `+0x19C` as a signed 32-bit count. For each visited candidate, object `+0x1A0` is reread, and the candidate’s 32-bit name pointer is fetched from that current table.

F-U02 is called with the retained slot input pointer and the fetched candidate pointer, in that order.

- A nonzero return advances to the next candidate, subject to a fresh count read.
- A zero return selects that candidate and prevents further candidate visits for the slot.

Selection is defined only by this observed zero return. The evidence does not establish semantic equality, reflexivity, stable results, or even that the dependency is a conventional string comparator.

For a selected candidate:

1. Reread object `+0x17C` and write the low 16 bits of the candidate index to that slot.
2. Separately reread object `+0x180` and write the saved candidate pointer to that slot.
3. Call F-U02 with the distinguished `.notdef` pointer first and the saved candidate pointer second.
4. If this second call returns nonzero, incorporate the slot index into the bound accumulators.
5. End the candidate search for that slot.

The candidate pointer fetched before the first comparison is retained across that call and used for both the selected-pointer store and the second comparison. It is not refetched from a potentially changed table.

The two selected-output stores use freshly read output-array pointers after the first comparison. They need not target the same arrays used for the defaults.

#### Direct writes versus retained mapping state

For a null input, this function performs only the two default mapping writes for that slot. For a nonnull input with no zero-returning candidate comparison, it likewise performs no selected-output writes.

Neither case guarantees that the defaults remain stored. Unsuccessful comparison calls may alter output arrays or object-held array pointers. Comparisons for later slots may also alter earlier entries. The second comparison can alter the selected entries after this function writes them.

Likewise, writing a candidate pointer does not guarantee that pointer remains in the destination afterward. These are direct-write guarantees, not persistent-state guarantees.

#### Bounds and final publication

A selected slot contributes to bounds exactly when its second comparison returns nonzero. A zero return excludes it, regardless of the strings’ apparent contents.

For a contributing slot, the minimum changes only if its index is less than the current minimum; the maximum changes only if its index exceeds the current maximum.

After the outer loop, if maximum is less than minimum, minimum becomes `0` and maximum receives a fresh read of workspace `+0x74`. That fallback count need not equal any earlier count.

The function then writes:

1. Minimum to object `+0x174`.
2. Maximum to object `+0x178`.
3. Workspace `+0x60` to object `+0x170`.

A tracked maximum is an inclusive slot index. The fallback maximum is instead the current count. The minimum sentinel is observable: if every contributing slot exceeds `32000`, the stored minimum remains `32000`.

### 6. Final cleanup

Every ordinary control-flow path calls F-ac8d8ec6121c with the workspace address and then returns the retained status. The cleanup return value is ignored.

No visible direct operation restores object fields or reverses dependency effects. No separate release operation for the scratch buffer is shown.

## Side effects

Direct effects include object mutation, writes through mapping-array pointers, workspace mutation, and scratch-buffer initialization.

Dependency effects are not restricted by this contract. In particular, dependencies are not assumed to preserve globals, object fields, reachable arrays, string contents, or workspace values. Allocation, retention, and release behavior remain unspecified.

Important distinctions:

- Initial object writes occur even on first-stage failure.
- Status `3` does not prevent subsequent publication or mapping.
- Skipping mapping means no direct mapping-array or mapping-result writes by this phase; it does not imply those locations are unchanged by dependencies.
- Mapping stores pointer values rather than copying string contents.
- Final cleanup occurs after publication and mapping, so final retained object state is also subject to that dependency’s unknown effects.
- Repeated reads do not establish thread safety.

## Dependencies

| Opaque ID | Established use |
|---|---|
| F-fb875b7a8097 | First gated call; receives workspace and three object-derived values |
| F-973fbfc6bac0 | Two gated passes sharing workspace and scratch storage |
| F-6a5c025e65c4 | Gated operation between those passes |
| F-aa5254170a3e | Metadata refresh; result ignored |
| F-4457a9a44a68 | Diagnostic on zero workspace `+0xDC`; result does not determine status |
| F-ac8d8ec6121c | Final workspace cleanup; result ignored |
| F-U01 | Workspace preparation with size `0x174`; full semantics unknown |
| F-U02 | Mapping comparison calls; only zero/nonzero branching established |

F-U01 and F-U02 are document-local opaque identifiers for dependencies without supplied mapped IDs. They assert no implementation identity beyond their described call sites.

## Acceptance examples

These are conditional static acceptance cases, not runtime results. Each assumes valid reached accesses and the stated dependency behavior.

1. **First-call failure.** If F-fb875b7a8097 returns a nonzero status, the function calls cleanup and returns that status. Initial object writes have occurred; the scratch buffer has not been explicitly cleared.

2. **Second-pass failure.** If the first three gated calls return zero and the second processing pass returns nonzero, flag clearing, metadata inspection, publication, and mapping are skipped.

3. **Absent metadata.** A null metadata pointer at the initial inspection causes no metadata-refresh call.

4. **Invalid first header word.** A nonnull metadata pointer whose first word is zero causes refresh without reading the second word for initial validity. A nonnull post-refresh pointer enables scanning without another header-validity test.

5. **Missing character-string data.** Zero workspace `+0xDC` triggers the diagnostic and status `3`; later copies and selector-controlled mapping still execute.

6. **First zero comparison.** Suppose a slot’s candidate comparisons return nonzero at indices `0` and `1`, then zero at `2`. The function selects index `2`, writes its low 16 bits and saved pointer using the then-current output pointers, and does not visit candidate `3` for that slot.

7. **Pointer replacement during comparison.** If the selecting comparison changes both output-array pointers, the selected stores use the replacement pointers. They still store the candidate pointer fetched before that comparison.

8. **Changed candidate table.** If a nonzero comparison replaces object `+0x1A0` and increases `+0x19C`, the next iteration tests the new count and fetches through the new table. The slot’s input pointer remains the previously fetched value.

9. **No contributions.** If no selected candidate has a nonzero second-comparison return, bounds fall back to `0` and the final reread slot count. This conclusion does not depend on semantic name equality.

10. **Wide selected index.** A selected candidate index of `65536` produces a 16-bit zero index write while the name-pointer write uses that candidate’s saved pointer.

11. **Nonpositive stable slot count.** If the slot count is nonpositive and unchanged, no slots are visited; fallback bounds are `0` and that count.

## Uncertainties

The following remain **documented_only**:

- Full contracts and side effects of all dependencies.
- Numeric values of the two static initialization words.
- Whether workspace preparation initializes every subsequently read field.
- Ownership, lifetime, and release requirements for published values and pointers.
- Comparison semantics beyond the observed zero/nonzero decisions.
- Valid count ranges, capacities, pointer alignment, and string-storage requirements.
- Safe behavior under invalid storage, harmful aliasing, concurrent mutation, or overflowing address arithmetic.

Addresses and principal scalar fields are 32 bits. Mapping indices narrow to 16 bits. Mapping loop comparisons use signed 32-bit counts; metadata scanning uses an unsigned 32-bit count. Address strides are 2, 4, and 12 bytes. No visible bounds checks establish that these calculations remain within supplied storage.

## Evidence

This contract derives solely from the supplied static function body and dependency-ID mapping.

Established observations include explicit access widths and offsets, initialization order, short-circuit call gates, metadata rereads, publication order, mapping count and pointer rereads, retained comparison arguments, index narrowing, bound sentinels, status assignment, and final cleanup placement.

Font-loading purpose and resource-transfer conventions are inferred intent, not established contracts. Dependency implementations were unavailable. No runtime tests have been performed.
