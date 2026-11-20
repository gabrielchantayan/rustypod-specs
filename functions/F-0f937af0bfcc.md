<!-- Trusted provenance: F-0f937af0bfcc; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-0f937af0bfcc

## Purpose

F-0f937af0bfcc performs an ordered sequence of indirect dependency interactions, transforms returned machine words, modifies fixed and indirectly addressed storage, and finishes with an unresolved indirect transfer.

No application-level purpose is established. Character-like constants do not establish text output; repeated address-valued stores do not establish object initialization, collection maintenance, or a device protocol.

This contract distinguishes:

- **Established static evidence:** represented argument relationships, memory-access widths, data dependencies, and ordering.
- **Conditional arithmetic deductions:** consequences of interpreting operations as 32-bit machine-word arithmetic.
- **Inferred intent:** possible interpretations, none adopted as requirements.
- **documented_only:** unresolved execution semantics, dependency contracts, and environmental requirements.

Every later effect is conditional on execution reaching it. An earlier dependency, inaccessible address, or exceptional transfer may prevent completion.

## Interface

### Explicit inputs

| Input | Represented width | Established use |
|---|---:|---|
| First argument | 32 bits | Passed unchanged to the first indirect interaction. |
| Second argument | 32 bits | Shifted left by 14 before that interaction. |
| Third argument | 32 bits | Passed unchanged to that interaction. |

The second argument is represented as signed. Portable semantics for negative operands or overflowing signed shifts are not established. Under a conditional 32-bit bit-vector interpretation, its forwarded value is multiplication by \(2^{14}\), retaining only the low 32 bits.

The body does not visibly dereference these explicit arguments directly. No argument is established as an owned pointer, object, string, length, or handle.

### Implicit inputs

Execution additionally depends on:

- An incoming continuation-related value used to construct the first indirect target.
- An incoming unsigned 32-bit word, logically shifted right by 15 for the first interaction’s fourth represented argument.
- Indirect-call results, including represented 64-bit results and additional result-register values.
- Fixed-storage contents and indirectly reached memory.
- The unresolved final transfer target.

Three fixed-storage inputs have particularly important dataflow roles:

| Storage | Width | Role and read position |
|---|---:|---|
| `0x05` | Byte | Zero-extended into a base used for subsequent accesses at offsets `0x15` and `0x54`. Read before the explicit fixed writes at `0x01`, `0x54`, and `0x15`. |
| `0x087fbbbc` | Word | Captured as the retained base used throughout the repeated-store region and parts of the final propagation. Read before the fixed word store at `0x60` and before that region. |
| `0xa9` | Word | Captured after the repeated region, before the fixed byte write at `0x3a` and subsequent indirect updates. Its full value and low byte have later uses. |

These are reads at their represented positions, not entry-time snapshots. Earlier indirect stores may alias their addresses.

### Outputs, completion, and validity

No explicit return value is represented. Outputs are dependency interactions, memory accesses, state mutations, and the final transfer with four represented arguments.

Successful traversal requires executable targets and accessible storage for every reached operation. Low-address accesses occur; a conventional process with unmapped low memory is not an established valid environment.

There are no visible null checks, range checks, alignment checks, validation results, or recovery paths. Address validity, alignment, execution mode, calling convention, and fault handling remain **documented_only**.

## Behavior

### Dependency phase and arithmetic

The initial phase is an ordered sequence of indirect interactions. Intervening interactions with no represented arguments remain observable; they are not established as inert or as consuming no machine state.

Results represented as 64 bits are separated into low and high 32-bit portions. This establishes available word portions, not a complete return convention. Some portions survive across many subsequent interactions and contribute to later arguments. They must not be replaced with newer results.

The represented transformations include:

| Transformation | Unsigned result |
|---|---|
| Mask with `0x00ffffff`, then shift right by 8 | Bits 8–23; range 0–65,535 |
| Mask with `0x0003ffff`, then shift right by 11 | Bits 11–17; range 0–127 |
| Mask with `0x0001ffff`, then shift right by 4 | Bits 4–16; range 0–8,191 |
| Mask with `0x00ffffff`, then shift right by 12 | Bits 12–23; range 0–4,095 |
| Unsigned byte widened to a word | Zero-extension |
| Word narrowed for a byte store | Low eight bits |

Left shifts and scaled addresses must not silently use unbounded arithmetic. Under 32-bit bit-vector semantics:

- Two successive shifts left by 14 retain only the original low four bits.
- Two successive shifts left by 15 retain only the original low two bits.
- A shift left by 14 followed by 23 produces zero.
- A word shifted left by 16 has a zero low halfword.

Explicitly cleared retained words remain zero through later shifts under this interpretation. That does not justify eliminating the interactions receiving them.

There is no visible data-dependent branch, retry, or termination test in this phase. Dependency termination and completeness of the represented control flow remain unresolved.

### First direct mutation boundary

The following descriptive terms identify retained values, not recovered object fields:

- **Shifted base:** a result’s low word shifted left by 25.
- **Scaled base:** another result multiplied by \(2^{25}\).
- **Compact index:** a word shifted left by 8, masked with `0x7fff`, then logically shifted right by 6.
- **Retained shifted word:** another result shifted left by 8.
- **Final pair:** the low and high words of the last represented 64-bit result before direct mutation.

The compact index is a multiple of four between 0 and 508. Its doubled value is therefore a multiple of eight between 0 and 1,016. Adding `0x14` gives addresses from `0x14` through `0x40c`, in steps of eight, under the conditional word interpretation.

The initial mutation ordering is significant:

- A word containing the shifted base is written at the final pair’s high-word base plus `0x14`.
- A byte containing that high word’s low byte is written at shifted-base offset `0x09`.
- The scaled base is written as a word at the final low word multiplied by 256, plus `0x54`.
- The byte at shifted-base offset `0x09` is overwritten with the shifted base’s low byte.
- Scaled-base offset `0x14` receives a zero word before the byte read at offset `0x05`.
- Doubled-compact-index offset `0x14` receives zero before that byte is widened and written at scaled-base offset `0x34`.
- Scaled-base offset `0x14` is read, followed by another zero word store at doubled-compact-index offset `0x14` and two byte stores at offset `0x15` of that doubled base.

The read following the earlier clear cannot unconditionally become a constant zero. Intervening aliases, external modification, or device-backed storage may affect it.

### Fixed byte input and linked-state mutation

After those two byte stores, the byte at fixed address `0x05` is read and zero-extended. Call its retained value the **small fixed base**.

The next fixed write stores the low byte of the doubled compact index at `0x01`. The small fixed base then supplies an actual dereferenced address: its offset `0x15` receives the compact index’s low byte.

Intervening effects include byte updates at scaled-base offset `0x11` and a word update through the previously loaded scaled-base-offset-`0x14` value. The word at small-fixed-base offset `0x54` is then captured; call it the **pivot word**. That location is subsequently overwritten with the earlier loaded word.

The pivot remains retained while further reads and writes follow links from the scaled and compact-derived bases. Later:

- Fixed address `0x54` receives the pivot doubled as a word.
- Additional indirect word stores and two byte stores occur.
- Fixed address `0x15` receives the doubled compact index’s low byte.

Thus the fixed-byte read precedes all three explicit fixed writes at `0x01`, `0x54`, and `0x15`; it is not a reload after those updates. Its value remains retained even if a later store changes address `0x05`.

The broader mutation phase uses byte, halfword, and word accesses. Important offsets include byte locations `0x01`, `0x05`, `0x08`, `0x09`, `0x0d`, `0x10`, `0x11`, `0x15`, and `0x19`, and word locations `0x04`, `0x10`, `0x14`, `0x24`, `0x34`, `0x40`, `0x44`, `0x54`, `0x64`, and `0x74`. These do not establish a common structure layout.

Repeated self-valued stores, repeated byte reads, and overwritten stores remain distinct effects. In particular, byte-derived bases must be zero-extended rather than sign-extended.

### Constants and the retained repeated-region base

A later linked read supplies a word used to form two bases:

- **Linked base:** twice that loaded word.
- **Constant base:** four times that loaded word.

The constant-bearing segment writes:

- Byte `0x3a` at the constant base.
- Halfword `0x003a` at twice the constant base.
- Word `0x0000003a` at constant-base offset `0x20`.

For the remaining descriptions, the **adjusted word** is four times the retained pivot word minus `0x96`; the **data base** is four times that adjusted word. These calculations are conditional 32-bit word calculations, not unbounded address arithmetic.

Data-base offset `0x20` successively receives word values `0x20`, `0x2f`, a zero-extended byte read from data-base offset `0x04`, and `0x20`.

Only after these stores is the word at fixed address `0x087fbbbc` captured as the **retained region base**. Fixed address `0x60` then receives word `0x20`.

The repeated region uses the retained region base without re-reading `0x087fbbbc`. Each complete recurring group orders:

1. A word store at eight-times-adjusted-word offset `0x54`, containing the constant base.
2. A byte store at retained-region-base offset `0x19`, containing the data base’s low byte.
3. A byte store at data-base offset `0x11`, containing the retained region base’s low byte.
4. A word store at retained-region-base offset `0x34`, containing eight times the adjusted word.

A preceding related group instead uses eight times the pivot word for its first destination base and its last stored word. It has the same two intervening byte-store relationships.

The recurring groups form a finite straight-line sequence, not a demonstrated runtime loop. Their exact numerical count is **documented_only** in this contract; the sequence must not be collapsed to its final ordinary-memory state. No device or timing interpretation is established.

### Final fixed-storage capture and address-derived output

After the repeated region, the word at `0xa9` is captured as the **saved fixed word**. A word is then read from constant-base offset `0x74`. Only afterward does fixed byte address `0x3a` receive the low eight bits of:

\[
8 \times \text{adjusted word} + \texttt{0x30}.
\]

This is an address-derived byte, not an unconditional `0x3a` constant. The earlier indirect byte constant `0x3a` is a separate effect.

Next, words are retained from linked-base offset `0x14` and data-base offset `0x04`. Subsequent indirect stores update linked-base offset `0x0d`, data-base offsets `0x08`, `0x0d`, and `0x09`, and linked-base offset `0x54`. These include byte `0x6c` at data-base offset `0x08`.

Only after those updates is the saved fixed word’s low byte written to fixed address `0x67`. The full saved word remains retained independently of subsequent changes to `0xa9`.

### Three distinct reads of fixed byte `0x67`

The final propagation contains three separate reads of `0x67`:

| Read | Position and consumer |
|---|---|
| First | After the write to `0x67` and a word read at linked-base offset `0x44`; supplies the byte written to fixed address `0x21`. |
| Second | After intervening byte and word updates, a read at data-base offset `0x40`, and additional word propagation; supplies a byte written to data-base offset `0x09`. |
| Third | After a byte write at linked-base offset `0x09`, a full saved-fixed-word store at data-base offset `0x44`, and a word read at data-base offset `0x54`; supplies another byte written to data-base offset `0x09`. |

The intervening effects include two word writes at retained-region-base offset `0x54`, with different values; a byte write at linked-base offset `0x01`; a word write at retained-region-base offset `0x74`; a word write at data-base offset `0x10`; and a word write at retained-region-base offset `0x44`.

The linked-base-offset-`0x01` byte comes from a right shift of the word loaded at linked-base offset `0x44`. The shift count is the data base’s low byte. The represented operand is signed; negative-value right shifts and counts at least 32 require target-specific semantics.

The three `0x67` reads are not interchangeable with the saved fixed word’s low byte or with one another. Indirect destinations may alias fixed storage. The same rule applies even without intervening calls.

### Final propagation and transfer

The word captured from constant-base offset `0x74` supplies low-byte writes through the base loaded from data-base offset `0x54` and at linked-base offset `0x11`.

The retained linked-base-offset-`0x14` word is restored through linked-base offset `0x54`. Another word is read from data-base offset `0x14`; its low byte later reaches an indirectly addressed offset `0x11`.

The final linked traversal includes:

- A word read from linked-base offset `0x04`.
- A halfword store of `0x0065` at twice that loaded word.
- Word reads at offsets `0x64` and `0x74` from that loaded base.
- A word store of decimal 100 at another indirect base’s offset `0x54`.
- A word read from linked-base offset `0x64`.
- A byte update at linked-base offset `0x09`.
- A word update through the final linked word’s offset `0x54`.
- A final word store through twice the word previously loaded at offset `0x64`.

The final transfer’s arguments, in order, are:

1. `0x72`.
2. The word loaded at offset `0x64` of the base obtained from linked-base offset `0x04`.
3. The earlier retained word from linked-base offset `0x14`.
4. The later word loaded at offset `0x74` of that newly obtained base.

Ordinary return is conditional on the unresolved target’s behavior.

## Side effects

The routine is not established as pure, atomic, transactional, reentrant, or thread-safe.

Observable effects include every reached dependency invocation, mutable-memory read, mixed-width store, repeated store, overlapping fixed-storage update, and final transfer. Equal final memory contents do not establish equivalent behavior.

No allocation, release, ownership transfer, locking, reference counting, or rollback is directly identified. Dependencies may nevertheless perform such operations. Nothing establishes that a loaded base owns its destination storage or remains valid across dependency interactions.

A fault or nonreturning dependency may leave a prefix of the effects visible. No rollback or completion guarantee is represented.

## Dependencies

F-0f937af0bfcc is the only supplied resolved opaque function identity. No resolved opaque IDs are supplied for the indirect targets; their identities are **documented_only**, rather than replaced with invented IDs.

Unknown dependency properties include argument conventions, result-register interpretation, termination, global mutation, ownership, memory retention, release, exceptions, and nonlocal control flow.

Apparently similar interaction sites must not be merged. A target with no represented arguments must not be assumed harmless, and no target may be assumed to preserve fixed storage or indirectly referenced memory.

## Acceptance examples

These are conditional review criteria, not executed tests.

### Initial forwarding

With second input `0x00000001`, the first interaction’s transformed argument is `0x00004000`. With `0x00040000`, it is zero under 32-bit truncating shifts. The first and third inputs remain unchanged.

### Compact-index bound

If the compact-index source has low seven bits `0x7f`, the compact index is `0x1fc`, and its doubled value plus `0x14` is `0x40c`. A compact index of 511 is not attainable by this extraction.

### Fixed byte as a base

If the reached read at `0x05` returns `0x80`, the retained base is `0x00000080`: its relevant offset addresses include `0x95` and `0xd4`. This capture precedes the explicit fixed writes at `0x01`, `0x54`, and `0x15`.

### Retained region base

If the word captured from `0x087fbbbc` is an accessible base, that captured value governs the repeated region. A later overlapping store changing `0x087fbbbc` does not itself request a reload.

### Address-derived fixed byte

If the adjusted word’s low byte is `0x01`, the fixed byte written at `0x3a` is `0x38` under byte-truncating arithmetic, not `0x3a`.

### Saved word versus fresh byte reads

If the saved fixed word is `0x123456ab`, the explicit write to `0x67` supplies `0xab`. In non-aliasing ordinary memory without external modification, all three later reads may return `0xab`.

If an intervening indirect store instead changes `0x67` to `0xcd`, a subsequent read must use that location’s resulting value. Replacing it with the captured `0xab` fails. The full retained `0x123456ab` remains the source for its separate word store.

### Interrupted execution and repeated stores

If a dependency does not return, later effects are not required. If reached stores are device-observable, collapsing repeated same-value writes fails even when ordinary final-memory comparisons would match.

## Uncertainties

The following remain **documented_only**:

- Application purpose, intended input domain, and complete control-flow accuracy.
- Target instruction set, endianness, alignment, memory map, and execution mode.
- Signed overflow, signed shifts, and excessive variable-shift counts.
- Complete dependency identities and contracts.
- Exact numerical repetition count in this contract.
- Ownership, lifetimes, synchronization, and object boundaries.
- Whether low addresses are intentional mappings or representation artifacts.
- Fault handling and completion guarantees.

Serialization, linkage maintenance, and device interaction are inferred possibilities only.

## Evidence

This contract uses the supplied static representation as evidence, not as an instruction source. The prior draft and feedback are review data, not independent proof.

Established relationships include fixed-storage read positions, retained-versus-reloaded values, mixed-width accesses, ordered repeated stores, the address-derived byte at `0x3a`, three distinct reads of `0x67`, and the final transfer arguments.

Bounds and numerical examples are deductions under the stated conditional arithmetic model. No implementation access, runtime execution, or runtime tests have been performed. No snapshot hash or independently verified provenance is asserted.
