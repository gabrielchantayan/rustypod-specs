<!-- Trusted provenance: F-eb53a0fc923a; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-eb53a0fc923a

## Purpose

F-eb53a0fc923a performs an ordered chain of unresolved indirect calls, transforms their represented results, mutates memory through calculated and fixed addresses, and invokes a final unresolved target.

The available static evidence establishes this structure, not an application-level purpose. It does not establish text formatting, object initialization, collection management, or a protocol. Constants resembling character codes are stored at different widths and unrelated calculated destinations; they do not establish a string layout.

This contract distinguishes:

- **Established static evidence:** represented operations, widths, data dependencies, and ordering.
- **[INFERENCE]:** consequences requiring an assumed machine interpretation.
- **documented_only:** unresolved behavior or a detail not fully specified here.

All execution descriptions are conditional on reaching the relevant operation. No runtime tests have been performed.

## Interface

### Explicit inputs

| Input | Represented width | Established use |
|---|---:|---|
| First argument | 32 bits | No explicit reference in the displayed body. |
| Second argument | 32 bits | No explicit reference in the displayed body. |
| Third argument | Unsigned 32 bits | Supplies two early shifted arguments to indirect dependencies. |

The two third-argument transformations are its unsigned 32-bit value shifted left by 24, and its low eight bits shifted left by five. Both depend only on the low byte under 32-bit arithmetic.

Absence of explicit references to the first two arguments does not establish that dependencies cannot observe their incoming machine state. Early indirect interfaces are incomplete.

### Implicit inputs

Execution additionally depends on:

- Incoming control-link state used to calculate the first call target.
- An incoming machine word shifted left by three.
- Primary and secondary results of unresolved calls.
- Fixed-location memory and memory reached through calculated addresses.
- An unresolved terminal target.

The function therefore cannot be specified solely as a mapping from its three explicit arguments.

### Outputs and completion

No explicit return value is represented. Observable outputs comprise dependency effects, memory accesses, and the terminal invocation.

The terminal invocation has four represented arguments:

1. `0x72`.
2. A 32-bit word loaded late from offset `0x64` of a calculated base.
3. A 32-bit word saved earlier in the late phase from offset `0x14` of another base.
4. A later 32-bit word loaded from offset `0x74` of the base supplying the second argument.

These are not established string parameters or ownership-bearing handles. An ordinary return follows the represented invocation, but whether the target returns is **documented_only**.

### Preconditions

Each reached call target must support the actual calling convention, and each reached address must support its access width. Required hidden machine state must be supplied by the environment.

No public object type, allocation extent, alignment guarantee, non-aliasing rule, or validation policy is established. Low numeric addresses are not implicitly valid, nor are they automatically null-pointer cases.

## Behavior

### Dependency-driven phase

A long straight-line sequence of indirect calls precedes the first displayed direct store. No conditional branch or loop is displayed within this sequence.

Calls with no displayed arguments alternate with calls receiving shifted results, selected bit fields, secondary result words, explicit zeros, and duplicated values. A call whose displayed result is unused can still have observable effects.

Most targets use a recurring address-offset construction; the first also depends on incoming control-link state. This construction does not establish a common callee, recursion, a dispatch-table API, or equivalent behavior between calls.

Represented 64-bit results are repeatedly separated into low and high 32-bit words. This establishes word extraction in the representation, not a confirmed language-level 64-bit return convention. Multiple machine-result registers remain a possible interpretation.

Call order must be retained. Nothing establishes that these dependencies preserve globals, preserve memory ownership, return normally, or leave earlier addresses valid.

### Arithmetic and widths

Unsigned extraction and truncating storage must not be replaced by unbounded arithmetic.

| Represented transformation | Result range |
|---|---:|
| Select low 20 bits, then logical right shift by ten | 0–1023 |
| Select low 24 bits, then logical right shift by twenty | 0–15 |
| Select low 19 bits, then left shift by two | 0–2,097,148, multiples of four |
| Select low 17 bits, then logical right shift by four | 0–8191 |
| Select low 18 bits, then logical right shift by eleven | 0–127 |

**[INFERENCE]** With truncating 32-bit machine arithmetic, left shifts and multiplications retain the low 32 result bits. A word shifted left by eight and then logically right by eight retains only its original low 24 bits. Two successive left shifts by 28 produce zero.

Signed-looking shifts, negative operands, overflowing signed-looking multiplication, and address wraparound require confirmed instruction semantics. Their exact machine treatment remains **documented_only**.

Byte stores retain eight bits; halfword stores retain sixteen bits. Explicit byte reads used as unsigned values are zero-extended when subsequently stored as 32-bit words.

### Direct-access phase and retained values

For clarity, this contract uses descriptive roles rather than inferred object names:

- The **wide base** is formed by multiplying a dependency result by `0x02000000`.
- The **compact base** is twice a field extracted by shifting another result left by eight, selecting fifteen low bits, and shifting right by six.
- The **saved anchor** is a word read later at offset `0x54` from the base obtained through fixed location `0x05`.
- The **retained global base** is loaded once from fixed location `0x087FBBBC`.
- The **adjusted value** is four times the saved anchor minus `0x96`.
- The **publication base** is four times the adjusted value.
- The **late doubled base** is twice a word obtained through a later byte-directed lookup.

These roles describe value provenance only. They do not imply records, allocations, ownership, or distinct memory regions.

**[INFERENCE]** Under 32-bit arithmetic, the wide-base multiplication retains seven low input bits. The compact base is a multiple of eight between 0 and 1016. Some direct accesses can consequently address low numeric locations.

### Early mutations and alias-sensitive reads

The opening direct-access region includes a word store through the high word of the last paired result, followed by a byte store through an earlier shifted result. A separate word store intervenes before that byte destination is written again with another value.

A zero word store at wide-base offset `0x14` precedes a byte read at wide-base offset `0x05`. A zero word store at compact-base offset `0x14` precedes storage of the zero-extended byte at wide-base offset `0x34`.

The word at wide-base offset `0x14` is then read and retained. Another zero store at compact-base offset `0x14` precedes two identical byte stores at compact-base offset `0x15`.

The retained word is not unconditionally zero: earlier calculated accesses may overlap, and memory-system behavior is unresolved. Repeated stores remain separate represented events.

Subsequent accesses use offsets including `0x04`, `0x14`, `0x44`, `0x54`, `0x64`, and `0x74`. Loaded words become both data and addresses. Byte reads at offset `0x05` also select later word-read addresses.

The function does not take an initial snapshot. Saved values remain saved, while later reads obtain memory at their own positions in the access sequence.

### Constants and repeated publication

A later region stores `0x3A` as a byte at four times a loaded word, as a halfword at eight times that word, and as a word at offset `0x20` from the first destination.

At publication-base offset `0x20`, the word-store sequence is `0x20`, `0x2F`, a zero-extended byte read from publication-base offset `0x04`, and `0x20`. That byte read follows the second store and precedes the third.

The retained global-base read follows this sequence. The fixed word write of `0x20` at `0x60` follows the base read and precedes the repeated publication region.

Each publication group has this ordered structure:

- A word store at offset `0x54` from a scaled calculated address.
- A byte store at retained-global-base offset `0x19`, using the publication base’s low byte.
- A byte store at publication-base offset `0x11`, using the retained global base’s low byte.
- A word store at retained-global-base offset `0x34`.

The initial group uses eight times the saved anchor for its scaled address and final word value. Subsequent groups instead use eight times the adjusted value. Their first word stores carry the same retained value derived from the earlier lookup.

The groups are visibly repeated straight-line operations, not a displayed runtime loop. Their total count is not quantified in this contract and remains **documented_only** here. This does not permit replacing the region with one group or its final memory image.

### Late mutation and completion

After publication, the function saves the word at fixed location `0xA9` and another word at offset `0x74` from the retained four-times-lookup value. It then writes fixed byte location `0x3A` before saving words from late-doubled-base offset `0x14` and publication-base offset `0x04`.

The next mutations include publication-base byte stores of `0x6C` at offset `0x08`, and the retained global base’s low byte at offsets `0x0D` and `0x09`. A word store at late-doubled-base offset `0x54` precedes publication of the saved `0xA9` word’s low byte at fixed location `0x67`.

The following fixed-byte rereads are specified under Side effects. They are interleaved with calculated stores and cannot be replaced by a saved byte.

A word loaded from late-doubled-base offset `0x44` also supplies a byte store derived from a signed-looking right shift. Its shift count is the publication base’s low eight bits. Counts can exceed 31; masking the count or rejecting such inputs is not established.

Later operations include:

- A word read at publication-base offset `0x40`, followed by a word store of that value at offset `0x10`, where a byte was previously stored.
- Writes using both the saved `0xA9` word and a freshly loaded destination word at publication-base offset `0x44`.
- A halfword store of `0x65` at twice a loaded word.
- A word store of decimal 100 at a calculated offset `0x54`.
- Final link-like word stores through loaded addresses.

The terminal invocation follows these accesses. No rollback, compensating write, or cleanup sequence is shown.

## Side effects

### Ordered fixed-location accesses

Fixed-location accesses are part of the same memory trace as calculated accesses. The following order is established; intervening calculated operations are significant.

| Position | Fixed-location behavior | Ordering and value relationship |
|---|---|---|
| Early direct-access region | Read at `0x05` | Follows the opening zero stores, retained word read, and duplicated compact-base byte stores. Its value is converted to unsigned 32 bits and retained as a calculated-access base. |
| Immediately afterward | Byte write at `0x01` | Stores the compact base’s low byte before the ensuing calculated stores and reads through the retained `0x05`-derived base. |
| Later early region | Word write at `0x54` | Stores the saved anchor shifted left by one. It follows a calculated store at compact-base offset `0x14` and precedes two further word stores of the saved anchor there. |
| After those anchor stores and two calculated byte stores | Byte write at `0x15` | Stores the compact base’s low byte. Further scaled-address mutations follow. |
| After the four-store constant sequence | Read at `0x087FBBBC` | Supplies the retained global base. The represented result is a 32-bit integer base. |
| Immediately after retaining that base | Word write at `0x60` | Stores `0x20`, before the first publication group. |
| After all publication groups | Word read at `0xA9` | Saved for later byte extraction and word storage. |
| After the following calculated word read at offset `0x74` | Byte write at `0x3A` | Stores the low eight bits of eight times the adjusted value plus `0x30`. |
| After the next saved reads and calculated mutations | Byte write at `0x67` | Stores the saved `0xA9` word’s low byte. |
| After a calculated word read at late-doubled-base offset `0x44` | Read `0x67`, then byte write `0x21` | The value read at this point supplies the write to `0x21`. |
| Later in the late phase | Read `0x67` for a calculated byte store | Supplies publication-base offset `0x09`. |
| After intervening calculated mutations and a word read | Read `0x67` again for another calculated byte store | Again supplies publication-base offset `0x09`, using the newly read byte. |

The read at `0x05` lacks a supplied global declaration establishing its access width. Conversion of its value to unsigned 32 bits does **not** prove a byte or word memory read. That width remains **documented_only**. The retained-base access is represented as producing a 32-bit base; its absent declaration does not independently confirm the underlying load width.

The byte and word classifications elsewhere above follow the represented assignments and explicit value widths. They do not establish atomicity or device-access semantics.

### Late fixed-byte rereads

The write to `0x67` does not create a reusable immutable byte value.

First, a calculated word read occurs between that write and the read supplying `0x21`. The write to `0x21` therefore uses the current byte read from `0x67`, not necessarily the byte originally published there.

Before the next read of `0x67`, calculated operations write bytes and words through the publication base, retained global base, and late doubled base. They also read publication-base offset `0x40`.

That next read supplies a byte store at publication-base offset `0x09`. Afterward, the function writes a byte at late-doubled-base offset `0x09`, stores the saved `0xA9` word at publication-base offset `0x44`, and reads a word at publication-base offset `0x54`. Only then does it reread `0x67` for the second byte store at publication-base offset `0x09`.

If any intervening calculated write overlaps `0x67`, the later read may differ. The two calculated byte stores are not guaranteed to carry equal values.

### Overlap and layout

The supplied representation explicitly warns that underscore-prefixed global symbols overlap smaller symbols at the same address. Such symbols must not be treated as disjoint storage merely because their names differ.

The warning establishes reported overlapping global views; it does not establish the complete overlap layout. Endianness, alignment behavior, atomicity, volatility, and memory-mapped-device behavior remain **documented_only**.

Calculated accesses can also overlap fixed locations and one another. Byte writes can affect later word reads, while word writes can affect later fixed-byte rereads. No object layout or non-aliasing guarantee is established.

### Ownership and errors

No explicit allocation or release operation is identified. Dependencies may nevertheless allocate, release, or otherwise change ownership; none of those behaviors is resolved.

No explicit bounds check, null check, error return, retry, or transactional recovery is represented. A failure may occur after earlier effects have become observable. Actual fault delivery and exception behavior remain **documented_only**.

## Dependencies

The supplied dependency map resolves only F-eb53a0fc923a itself. It provides no opaque function IDs for the indirect targets.

The unresolved dependencies comprise the entry-state-dependent first target, subsequent address-derived targets, and the terminal target. No invented function identities are assigned to them.

For each, the following remain **documented_only**:

- Complete parameter and result conventions.
- Global and calculated-memory effects.
- Allocation, release, ownership, and lifetime effects.
- Faults, exceptions, termination, and ordinary return behavior.

The recurring target construction does not prove recursion into F-eb53a0fc923a.

## Acceptance examples

These are conditional static criteria, not executed tests.

### Third-argument selection

With identical implicit state and dependency behavior, third arguments `0x000000AB` and `0x123456AB` produce the same two explicitly derived values: `0xAB000000` and `0x00001560`.

Whole-function equivalence is not established if dependencies observe additional incoming state.

### Width-sensitive extraction

For unsigned `0xFFFFFFFF`, selecting twenty low bits and logically shifting right by ten yields 1023, not −1.

For a represented paired result, a low word of `0x00000001` shifted left by eight yields `0x00000100`; an unshifted high word of `0x80000000` remains distinct.

### Retained global base

Suppose the read at `0x087FBBBC` produces a valid base. Publication uses that retained value even if the subsequent word write at `0x60` overlaps and changes the original global storage. Reloading the base after the write is not equivalent.

### Fixed-byte reread after aliasing

Suppose the saved `0xA9` word has low byte `0xA5`, but an intervening calculated store changes `0x67` to `0x3C` before the read supplying `0x21`. Under ordinary coherent byte-memory semantics, `0x21` receives `0x3C`, not the saved `0xA5`.

Likewise, if `0x67` changes between its two reads for publication-base offset `0x09`, those stores use their respective read values.

### Overlapping global views

If a word write through a reported overlapping global view changes a smaller symbol’s storage, later reads must reflect the actual memory interpretation. A model assigning independent storage to every global name does not satisfy the reported overlap condition.

### Access-trace preservation

Both early repeated byte stores, all four constant-sequence word stores, the intervening byte read, and the repeated publication groups remain separate represented events. Matching only final memory is insufficient.

### Terminal invocation

If all preceding operations complete, the terminal target receives `0x72` first and the three specified saved or loaded words afterward. If the target does not return, ordinary function return is not required.

## Uncertainties

The following remain **documented_only**:

- Application purpose and meaningful public input domain.
- Hidden calling-convention inputs and secondary-result conventions.
- Every indirect dependency’s identity and behavior.
- Signed-looking overflow, oversized shifts, and address arithmetic semantics.
- Widths of fixed reads not confirmed by supplied declarations.
- Exact publication-group count, not enumerated here.
- Complete overlapping-global layout and endianness.
- Address validity, alignment, memory type, concurrency, and atomicity.
- Ownership, lifetime, fault handling, and terminal-target return behavior.
- Fidelity of the static representation to actual machine control flow.

These limits do not justify treating the function as a no-op, formatter, or safe initializer.

## Evidence

The evidence is the supplied static representation for F-eb53a0fc923a and the supplied dependency map. The representation supports the explicit transformations, access widths where recoverable, fixed-location ordering, retained-base use, repeated writes, late byte rereads, and terminal invocation described here.

The overlapping-global warning is recorded as evidence from that representation, not as an independently validated complete memory layout. Prior prose and reviewer feedback do not independently establish behavior.

No dependency implementations, confirmed instruction semantics, runtime traces, or validated object layouts were available. No runtime tests have been performed. Arithmetic consequences requiring truncating 32-bit machine semantics are marked **[INFERENCE]**; unresolved behavior remains **documented_only**.
