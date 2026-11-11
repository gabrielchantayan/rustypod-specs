<!-- Trusted provenance: F-eda87ef38dc8; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-eda87ef38dc8

## Purpose

F-eda87ef38dc8 replaces a 16-byte state in place using byte-valued lookups and a fixed permutation of source positions.

When the global values observed during address calculation agree, and under the 32-bit modular interpretation specified below, each destination selects a byte from its own 256-byte table. The selected entry is indexed by the source byte assigned to that destination.

The established behavior is sequential memory access, not an atomic transformation. Lookup storage need not be disjoint from the state. Earlier state writes can therefore affect later lookup reads.

A table-based substitution combined with a permutation is an inferred purpose. No particular algorithm, table contents, invertibility, or security property is established.

## Interface

### Inputs

- **State address:** identifies 16 consecutive byte locations, numbered 0 through 15. Every location is read once and overwritten once during successful completion.
- **Lookup base:** an integer-valued address used as the origin for computed one-byte reads. It is not a length or a pointer to a collection of table pointers.
- **Implicit global word:** contributes values to the address calculation. Its observations are interleaved with the first state access, as specified under Behavior.

### Output

There is no returned value or explicit status. Successful completion leaves the 16 replacement bytes in the supplied state storage.

### Widths and address interpretation

State values and lookup results are unsigned 8-bit quantities, ranging from 0 through 255. Lookup reads and state writes each concern one byte.

The supplied representation supports a 32-bit word interpretation, but independent declarations establishing every integer and pointer width are unavailable. Consequently, the address equations in this contract are conditional on the following model:

- Word arithmetic is modulo \(2^{32}\).
- Observed global values are interpreted as 32-bit word patterns.
- Address addition uses that same modular width before the resulting value is interpreted as an address.
- A residue “modulo 256” means the unique value from 0 through 255.

This model describes the address calculation mathematically. It does not establish portable source-language behavior for signed overflow, integer-to-pointer conversion, or invalid addresses.

### Storage, ownership, and preconditions

No allocation, release, ownership transfer, or retained reference is visible. The function does not establish who owns either storage region.

Ordinary successful execution requires:

- All 16 state locations to be readable and writable when accessed.
- Every computed lookup address to permit a one-byte read.
- The global word to be readable at each represented observation.
- The environment to support the required arithmetic and address interpretation.

Additional conditions apply to the simplified entry-state result:

- All represented global observations must agree.
- Lookup values must remain unaffected by preceding state writes and external mutation.
- State storage must remain unaffected by external mutation throughout the relevant state accesses.

The function performs no visible checks enforcing these conditions. In particular, a readable 4096-byte lookup region is sufficient for the ordinary table layout only under the equal-global address simplification; differing observations can select addresses outside that region.

## Behavior

### Global observations and the first state access

Three global-value references appear in the supplied representation. This contract names their values by their roles:

- \(G_{\mathrm{early}}\): the initial observation.
- \(G_{\mathrm{increment}}\): the observation whose value is incremented by one for subsequent calculations.
- \(G_{\mathrm{first}}\): the observation referenced during the first lookup-address calculation.

Their ordering relative to state access is significant:

1. Observe \(G_{\mathrm{early}}\).
2. Read state position 0 and derive its byte index.
3. Observe \(G_{\mathrm{increment}}\), then form the incremented word.
4. Reference \(G_{\mathrm{first}}\) during calculation of the first lookup address.
5. Read the lookup byte for destination 0.
6. Write that byte to state position 0.

Thus the initial global observation precedes the first state-byte read, while the observation used for the increment follows that read. The final represented global reference occurs before the first state write.

These are logical observations in the supplied representation. They do not establish three distinct physical load instructions, volatile semantics, or a supported concurrent-access protocol.

No further global reference is represented after the first lookup calculation. Later addresses use the already obtained values.

### Reduced address characterization

The following characterization covers both equal and differing global observations without retaining the original arithmetic structure.

Let:

- \(d\) be the destination position, from 0 through 15.
- \(x\) be the selected source byte as actually read or previously preserved.
- \(a\) be \(G_{\mathrm{first}}\) for destination 0, and \(G_{\mathrm{early}}\) for every other destination.
- \(b\) be \(G_{\mathrm{increment}}\), meaning the observed value **before** its increment.

The lookup address is the lookup base plus

\[
256d+x+\Delta,
\]

reduced modulo \(2^{32}\).

The correction \(\Delta\) has two cases.

**For source bytes 0 through 93:**

\[
\Delta=b-a.
\]

This is a word-sized correction and is independent of the particular source byte within that range.

**For source bytes 94 through 255:**

Define a byte-valued phase

\[
p=162(x-94)\bmod256.
\]

Let \(r_a\) and \(r_b\) be the residues modulo 256 of \(a-p\) and \(b-p\), respectively. Then

\[
\Delta=(a-b)+2(r_b-r_a).
\]

The residue difference is an ordinary integer between −255 and 255; the final correction and address are interpreted modulo \(2^{32}\).

This reduced rule establishes several distinctions:

- Differing global observations do not generally produce one uniform displacement for all source bytes.
- The correction can depend on low-byte wraparound as well as the full word difference.
- The effective address need not remain inside the destination’s ordinary 256-byte table.
- The correction for the first destination can differ from corrections used later because its choice of \(a\) differs.

### Ordinary table layout

When \(a=b\), both cases give \(\Delta=0\). In particular, if

\[
G_{\mathrm{early}}=G_{\mathrm{increment}}=G_{\mathrm{first}},
\]

every lookup address reduces to

\[
\text{lookup base}+256d+x.
\]

The layout is therefore 16 consecutive tables, each containing 256 one-byte entries. Table number equals destination position; entry number equals the selected source byte. The entire ordinary region occupies offsets 0 through 4095.

There are no visible table headers, indirect table pointers, or gaps between entries.

The byte-index preprocessing wraps between source values 93 and 94, but this does not rotate the effective index when the global observations agree.

### Source selection

The destination-to-source mapping is:

| Destination | Source position |
|---:|---:|
| 0 | 0 |
| 1 | 5 |
| 2 | 10 |
| 3 | 15 |
| 4 | 4 |
| 5 | 9 |
| 6 | 14 |
| 7 | 3 |
| 8 | 8 |
| 9 | 13 |
| 10 | 2 |
| 11 | 7 |
| 12 | 12 |
| 13 | 1 |
| 14 | 6 |
| 15 | 11 |

The source-position permutation and destination-specific table choice are independent parts of the behavior.

**Entry-state interpretation requires all of the following:** agreeing global observations, lookup values unaffected by state writes or external mutation, and no external mutation of state storage throughout the relevant accesses. Under those conditions,

\[
\text{output}[d]
=
\text{lookup}\bigl[256d+\text{entry-state}[s(d)]\bigr],
\]

where \(s(d)\) is the source position in the table above.

Stable globals and stable lookup contents alone are insufficient to justify this formula: a later state-source read could otherwise observe an externally changed byte.

### Ordered state and lookup accesses

State writes occur in this order:

**0, 4, 8, 12, 1, 5, 9, 13, 2, 10, 6, 14, 3, 15, 11, 7.**

Each replacement lookup read precedes its corresponding state write. The significant access groups are:

| Group | Ordered memory behavior |
|---|---|
| Fixed positions | Following the initial global/state interleaving, complete destination 0. Then read, look up, and overwrite positions 4, 8, and 12 separately, in that order. |
| Cycle involving positions 1, 5, 9, and 13 | Preserve position 1 before reading position 5. Use position 5 for destination 1. Read position 9 for destination 5, then position 13 for destination 9. Finally use the preserved position 1 for destination 13. |
| Pair involving positions 2 and 10 | Read position 10, then preserve position 2, before the lookup and write for destination 2. Use preserved position 2 for destination 10. |
| Pair involving positions 6 and 14 | Preserve position 6, then read position 14, before the lookup and write for destination 6. Use preserved position 6 for destination 14. |
| Cycle involving positions 3, 15, 11, and 7 | Preserve position 3 before reading position 15. Use position 15 for destination 3. Read position 11 for destination 15, then position 7 for destination 11. Finally use preserved position 3 for destination 7. |

The function does not snapshot the complete state. It preserves selected source bytes to prevent its own permutation writes from destroying values needed later.

Successful completion includes 16 logical state-byte reads, 16 lookup-byte reads, and 16 state-byte writes. These counts do not specify machine instruction counts or physical memory transactions.

### Aliasing and mutation

If a selected lookup address aliases a state location already written, that lookup reads the location’s current byte, including any change caused by the earlier write. There is no lookup snapshot.

If external activity changes a state position before its scheduled read, the selected source value can differ from its entry value. Once a source value has been preserved, its later use does not reread the original location.

If state storage overlaps the global word, a state write can modify that underlying storage. Such a write occurs only after all represented global references. Later address calculations nevertheless use the previously obtained values; no later refresh is shown.

These ordering descriptions do not establish that concurrent mutation is defined or safe under any particular language memory model.

### Failure and partial completion

There is no visible validation, error branch, retry, rollback, or recovery path.

An invalid read or write can prevent completion according to the execution environment. Earlier state writes are not undone by any visible mechanism. The supplied material does not establish a particular exception, diagnostic, recoverable result, or failure return convention.

## Side effects

On successful completion:

- Every state position is overwritten exactly once, including when its replacement equals its previous value.
- One lookup byte is read for every destination.
- The global word is referenced as described above.
- No explicit write to lookup storage or the global word is shown, except for changes caused by aliasing with state storage.

No function invocation, allocation, release, synchronization operation, logging, or external I/O is visible.

Atomicity, memory barriers, thread safety, and protection against concurrent mutation are not established.

## Dependencies

No external function invocation is visible. There are no dependency-function IDs to enumerate.

The non-function dependencies are:

1. The caller-supplied state storage.
2. Memory reachable through the lookup-base address calculations.
3. The global word used in those calculations.
4. The environment’s integer, pointer, overflow, and memory-access semantics.

The global word’s initialization, permitted range, mutability, and relationship to other operations are unavailable.

## Acceptance examples

These are conditional static acceptance criteria, not executed test results.

### Identity-valued tables

Assume agreeing global observations, valid addresses, disjoint state and lookup storage, and no external mutation of either region during the relevant accesses. Every table entry contains its entry number.

For entry-state bytes 0 through 15 in positional order, the resulting state must be:

**0, 5, 10, 15, 4, 9, 14, 3, 8, 13, 2, 7, 12, 1, 6, 11.**

### Destination-specific constants

Under the same storage and global conditions, let every entry in table \(d\) contain byte \(d\).

Every output position \(d\) must contain \(d\), regardless of the entry-state bytes. Selecting a table according to source position instead of destination position would violate this criterion.

### Byte-index boundaries

With agreeing global observations, source bytes 0, 93, 94, and 255 select offsets:

- \(256d\)
- \(256d+93\)
- \(256d+94\)
- \(256d+255\)

The preprocessing wrap between 93 and 94 must not introduce an effective index discontinuity.

### Preserved sources

With state unaffected by external mutation and lookup contents unaffected by preceding writes, destination 13 uses the entry value of position 1, not its earlier replacement.

Destinations 10, 14, and 7 similarly use the preserved entry values of positions 2, 6, and 3.

### Differing first global observation

Assume the 32-bit model, \(G_{\mathrm{early}}=G_{\mathrm{increment}}=0\), and \(G_{\mathrm{first}}=1\).

If the byte read from position 0 is 0, the first lookup address is one byte before the lookup base, modulo \(2^{32}\). It is not the ordinary table-0 entry 0.

Later destinations use equal address-calculation globals and therefore retain their ordinary offsets. Successful completion still requires every selected address to be valid.

### Low-byte carry affects displacement

Assume \(G_{\mathrm{increment}}=255\), \(G_{\mathrm{first}}=256\), and the first source byte is 94.

The phase is 0, and the two low-byte residues are 0 and 255. The correction is therefore 511, making the first lookup offset **605**.

With those same global observations but a first source byte of 0, the correction is instead −1. A single displacement applied to every source byte would not satisfy the contract.

### Lookup overlap

Suppose a later selected lookup address aliases a state byte already written. If the earlier write changed that byte, the later lookup must read the changed value, absent another intervening modification.

An implementation that snapshots all lookup results at entry would not preserve this behavior.

## Uncertainties

- **documented_only — exact word and pointer widths:** The address rules are conditional on a 32-bit modular interpretation; independent platform declarations are unavailable.
- **documented_only — signed arithmetic:** The relationship between the displayed arithmetic and executable overflow semantics is not independently established.
- **documented_only — physical global loads:** Logical reference order is established, but physical load count, volatility, atomicity, and concurrency semantics are not.
- **documented_only — global lifecycle:** Initialization, intended invariants, and permitted changes are unknown.
- **documented_only — alias restrictions:** No storage-separation contract or bounds metadata is supplied.
- **documented_only — exceptional behavior:** Invalid accesses have no established recoverable result, diagnostic, or rollback behavior.
- **documented_only — table meaning:** Actual table contents and their intended interpretation are unavailable.
- **Inferred intent:** Substitution and permutation describe the operation’s shape, not an established algorithm or security guarantee.

## Evidence

Static inspection establishes the byte accesses, destination order, source preservation, global-reference interleaving, and absence of visible calls, validation, or recovery.

The piecewise address correction is an algebraically reduced characterization under the stated 32-bit modular model. The ordinary contiguous-table layout follows by setting the paired global observations equal; it is not an unconditional storage guarantee.

No runtime tests have been performed. No independent caller, global declaration, table initializer, or platform specification was available for corroboration.
