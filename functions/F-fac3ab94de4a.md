<!-- Trusted provenance: F-fac3ab94de4a; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-fac3ab94de4a

## Purpose

F-fac3ab94de4a performs an ordered sequence of mixed-width memory accesses, followed by an unresolved indirect control transfer carrying four argument values.

The established behavior includes twenty repetitions of a four-store pattern, subsequent accesses through incoming and memory-derived address bases, and direct accesses to four fixed-address locations. Its application-level purpose, object model, and successful-completion condition are unknown.

This contract describes the supplied static representation. It does not establish that all accesses are executable on a particular target, that the outgoing transfer is reached, or that execution returns normally.

## Interface

### Incoming values

The function has one explicit input and three additional incoming machine-state values:

| Contract designation | Represented width and interpretation | Uses |
|---|---|---|
| **P** | Apparently signed 32-bit integer | Stored word and address base |
| **A** | Unsigned 32-bit integer | Address arithmetic, stored words, byte values, and shift count |
| **B** | Apparently signed 32-bit integer | Address base and byte value |
| **C** | Apparently signed 32-bit integer | Address base and stored word |

These designations describe contract inputs, not application fields or source identifiers. The initialization and calling convention of A, B, and C are not established. Supplying P alone does not establish sufficient invocation state.

### Widths and arithmetic

A byte is 8 bits, a halfword is 16 bits, and a word is 32 bits. Offsets are measured in bytes. “Low byte” means the least significant eight bits, described as a stored bit pattern without assuming signed character interpretation.

Let **D** denote twice A, reduced modulo \(2^{32}\). Unsigned arithmetic involving A, including the initial destination \(D + 0x54\), has the corresponding modulo-\(2^{32}\) interpretation at the represented integer width. This does not establish that the resulting address is valid.

Address expressions involving P, B, C, or signed memory-derived values have unresolved signed-overflow boundaries. In particular, doubling a signed memory-derived base must not be assumed to wrap portably.

The representation uses integer values as address bases. The complete pointer conversion and target address-space rules are unknown.

### Output and preconditions

No explicit result value is represented. Observable behavior consists of memory accesses, possible memory changes or access-triggered effects, and the outgoing transfer.

Every reached access requires an appropriate mapping, permissions, and target-supported alignment. The function provides no visible null, bounds, alignment, permission, overflow, or target-validity checks.

No contiguous allocation, object size, or structure layout is established. An offset identifies an access location; it does not prove a field boundary or an allocation extent.

## Behavior

### Initial repeated stores

The initial phase represents twenty repetitions of this ordered pattern:

| Position within each repetition | Destination | Width | Stored value |
|---|---|---:|---|
| First | \(D + 0x54\) | 32 bits | P |
| Second | \(B + 0x19\) | 8 bits | Low byte of A |
| Third | \(A + 0x11\) | 8 bits | Low byte of B |
| Fourth | \(B + 0x34\) | 32 bits | D |

This phase contains eighty represented stores, with no intervening memory reads or visible branches.

The contract preserves repetition and order, not merely the resulting contents of ordinary RAM. Overlap or externally observable memory can make each access significant. Whether the represented stores become distinct hardware transactions is not established.

The incoming values remain held values throughout this phase. Writing to memory associated with an incoming value does not imply that the value is reloaded.

### Early captures and fixed-address accesses

After the repeated stores, the function captures a word from fixed address `0xA9`, then captures a word from \(P + 0x74\).

It next writes a byte at fixed address `0x3A`. The stored bit pattern is:

\[
(2 \times \operatorname{low8}(A) + 0x30) \bmod 256.
\]

Two further word captures follow, first from \(C + 0x14\), then from \(A + 0x04\). Later references to these captures use the saved values, not new reads of their original locations.

The complete explicitly fixed-address access pattern is:

| Address | Direct represented accesses |
|---|---|
| `0xA9` | One 32-bit read |
| `0x3A` | One 8-bit write |
| `0x67` | One 8-bit write, followed by three separate 8-bit reads |
| `0x21` | One 8-bit write |

Thus there are three explicitly written fixed-address locations, not four. The word read beginning at `0xA9` covers addresses `0xA9` through `0xAC`.

This table excludes accesses through computed addresses that happen to overlap these locations. It also does not describe effects internal to the unresolved dependency or side effects of reading a memory-mapped location.

### First linked updates

Following the early captures, the function writes the low byte of A at \(C + 0x0D\). It then writes byte `0x6C` at \(A + 0x08\), followed by the low byte of B at \(A + 0x0D\) and then \(A + 0x09\). A word store at \(C + 0x54\) contains C.

Next, the low byte of the saved word from `0xA9` is written at fixed address `0x67`. A signed word is then captured from \(C + 0x44\). Only after this capture does the function read the byte at `0x67` and write the observed byte at fixed address `0x21`.

The latter operation is a memory read followed by a store, not an unconditional reuse of the byte previously written at `0x67`. Equality between those values requires suitable memory and interference assumptions.

The signed word captured from \(C + 0x44\) supplies three subsequent stores, in order:

- Its low byte at \(A + 0x10\).
- Its full word at \(B + 0x54\).
- A shifted, byte-narrowed result at \(C + 0x01\).

The shift count is the low byte of A. The shifted operand is signed. For nonnegative operands and counts from 0 through 31, the represented right shift determines the result before byte narrowing. Negative operands with nonzero counts and counts of 32 or greater require semantics not established here. A zero count leaves the value unchanged before narrowing.

Immediately afterward, another word store at \(B + 0x54\) contains A. For ordinary writable memory, the later store supersedes the earlier contents at that location, subject to interference and access side effects. Both stores remain part of the represented sequence.

### Subsequent replacements and address capture

A word is next captured from \(A + 0x40\). The saved word from \(A + 0x04\) is stored at \(B + 0x74\), and the newly captured word is stored at \(A + 0x10\).

The latter store covers offsets `0x10` through `0x13` relative to A. In ordinary memory it replaces the earlier byte at \(A + 0x10\) and the initial-phase byte at \(A + 0x11\), regardless of aliasing between different bases.

The function then:

- Stores C as a word at \(B + 0x44\).
- Reads the byte at `0x67` and stores the observed byte at \(A + 0x09\).
- Stores the low byte of A at \(C + 0x09\).
- Stores the full saved word from `0xA9` at \(A + 0x44\).

A signed word read from \(A + 0x54\) establishes a new address base, designated **T**. This location is distinct as an expression from the initial repeated destination \(D + 0x54\); equality is not generally established.

After capturing T, the function again reads the byte at `0x67` and stores that observation at \(A + 0x09\). This is the third direct byte read of `0x67`. The three reads occur at different points and must not generally be treated as a single shared observation.

### Updates involving T

The low byte of the saved word from \(P + 0x74\) is written at \(T + 0x0D\).

The saved word from \(C + 0x14\) is then stored at \(C + 0x54\). In ordinary memory, this replaces the earlier store of C itself at that address, subject to intervening overlap and memory effects.

A word is next captured from \(A + 0x14\). After that capture, the function writes:

- The low byte of the saved \(P + 0x74\) word at \(C + 0x11\).
- The low byte of T at \(A + 0x09\).
- T as a word at \(A + 0x44\).

The explicit word stores at \(A + 0x44\) therefore first contain the saved fixed-address word and later T. These stores do not establish unconditional final contents: other computed destinations can overlap the same bytes, reads can have side effects, and the dependency may subsequently modify memory.

### Final memory-derived bases

A signed word read from \(C + 0x04\) supplies another address base, designated **U**.

The function writes the 16-bit value `0x0065` at twice U. It then reads a signed word from \(U + 0x64\), designated **V**. The halfword store precedes this read and may affect V if the accessed regions overlap.

Next, the low byte of the saved word from \(A + 0x14\) is written at \(T + 0x11\), followed by a word containing decimal 100 at \(T + 0x54\).

A word is then captured from \(U + 0x74\). After that, a signed word is captured from \(C + 0x64\), designated **W**.

The remaining stores before the outgoing transfer occur in this order:

| Destination | Width | Value |
|---|---:|---|
| \(C + 0x09\) | 8 bits | Low byte of U |
| \(W + 0x54\) | 32 bits | V |
| Twice V | 32 bits | W |

The word from \(U + 0x74\) has already been captured before these stores. Its outgoing use remains a saved value even if a later store overlaps the original read location.

The doubling of U and V uses signed values in the representation. Defined boundary behavior cannot be inferred from the modulo rule for A.

### Outgoing transfer

The unresolved indirect dependency receives four logical argument values:

| Position | Value |
|---|---|
| First | `0x72`, decimal 114 |
| Second | V |
| Third | Saved word from \(C + 0x14\) |
| Fourth | Saved word from \(U + 0x74\) |

The last three arguments originate as 32-bit word values. Their physical argument locations, extension rules, and complete machine-level calling convention are unknown.

An apparent return follows the invocation in the supplied representation. This does not establish that the dependency returns, that the transfer follows an ordinary call convention, or that the apparent return is reachable.

## Side effects

### Memory and observation

Memory writes occur through incoming bases, memory-derived bases, and three explicit fixed addresses: `0x3A`, `0x67`, and `0x21`.

There is no explicit fixed-address store to `0xA9` through `0xAC` before the indirect invocation. This is an access-description claim, not a contents-preservation guarantee. Computed stores can overlap that region, its read may be side-effecting, and the unresolved dependency may change it.

Aliasing can affect both data and subsequent address selection:

- A byte store may alter a later word read.
- A word store may replace multiple earlier byte stores.
- A store before an address-producing read may change the base used later.
- A captured value remains a snapshot despite later changes to its source memory.

Endianness is not established. Exact byte contents resulting from overlapping halfword and word accesses depend on the target byte order.

The represented order does not establish atomicity, synchronization, volatile semantics, hardware transaction widths, or inter-thread visibility. Read-sensitive memory may change state even without an explicit store to the same region.

### Ownership and failures

No allocation, release, reference-count operation, or explicit ownership transfer is visible before the indirect invocation. This does not establish the ownership behavior of the dependency or the resource meaning of any accessed memory.

There is no visible error result, recovery branch, rollback, or completion indicator. An invalid access may prevent later operations from being reached. No restoration of earlier effects is established.

The dependency must not be assumed to preserve globals, memory, machine state, or resource ownership.

## Dependencies

One unresolved indirect control-transfer dependency is visible. No opaque function ID is supplied for its target; the caller’s ID does not identify that target.

Its established interface is limited to the four logical argument values above. The following remain unknown:

- Target selection and identity.
- Calling convention and physical argument representation.
- Additional implicit inputs.
- Memory, global-state, and access-triggered effects.
- Ownership and lifetime requirements.
- Result, failure, and return behavior.

No resolved direct-call dependency is established.

## Acceptance examples

These are conditional deductions from static evidence, not executed tests.

### Fixed-address read versus write

Assume execution reaches the outgoing transfer and all computed store ranges avoid addresses `0xA9` through `0xAC`.

Before the indirect invocation, the function performs one explicit word read beginning at `0xA9` and no explicit store into those four bytes. Its explicitly fixed-address stores target only `0x3A`, `0x67`, and `0x21`.

This conclusion does **not** imply that the contents at `0xA9` through `0xAC` remain unchanged. The read may have memory-mapped side effects, and other access-triggered effects are unspecified. It also says nothing about dependency behavior after the invocation begins.

A contents-preservation conclusion for the pre-invocation interval additionally requires ordinary side-effect-free memory and absence of external modification. Extending that conclusion through the invocation would additionally require dependency preservation, which is not established.

### Repeated stores

Given valid destinations whose accesses are separately observable, the initial phase contains twenty ordered repetitions of the four-store pattern. Reducing it to one repetition may preserve final contents in ordinary nonoverlapping RAM, but does not preserve the represented access sequence.

### Byte arithmetic

If A’s low byte is `0xF0`, the byte written at `0x3A` is `0x10`: twice `0xF0` plus `0x30` is `0x210`, reduced modulo 256.

If B’s low byte is `0xAB`, each initial-phase byte store at \(A + 0x11\) contains `0xAB`. A later word store beginning at \(A + 0x10\) may replace that byte.

### Saved values

Suppose the word captured from \(C + 0x14\) is `0x12345678`, and execution reaches the relevant operations. The later word store at \(C + 0x54\) and the third outgoing argument both use that saved value.

Subsequent modification of the original read location does not itself change these saved uses.

### Distinct destinations

If A is nonzero and the relevant address arithmetic does not wrap, \(2A + 0x54\) differs from \(A + 0x54\). The repeated stores at the former therefore do not establish that the later read at the latter yields P.

### Shift boundaries

If A’s low byte is zero, the byte written at \(C + 0x01\) is the low byte of the signed word captured from \(C + 0x44\).

If A’s low byte is at least 32, no portable expected byte is established. Assuming a masked shift count would add an unsupported target rule.

### Outgoing arguments

If V is captured as `0x00002000`, the saved \(C + 0x14\) word is `0x11223344`, and the saved \(U + 0x74\) word is `0x55667788`, a reached invocation receives those values in this order:

`0x72`, `0x00002000`, `0x11223344`, `0x55667788`.

This establishes neither target identity nor post-invocation memory contents.

## Uncertainties

The following remain **documented_only**:

- Application purpose, field meanings, and object boundaries.
- Initialization and calling convention of the implicit incoming values.
- Target address semantics, byte order, and alignment requirements.
- Whether fixed or computed addresses refer to ordinary RAM, devices, or other mappings.
- Read-sensitive behavior and other access-triggered effects.
- Signed-overflow and negative-operand right-shift semantics.
- Out-of-range shift behavior.
- Hardware transaction multiplicity, atomicity, and visibility.
- Fidelity of the supplied representation to executable instructions.
- Dependency identity, target selection, ownership, and behavior.
- Reachability of the outgoing transfer and normal return.

Record initialization or linking is a possible interpretation, but only **inferred intent**. The access pattern does not establish such a data model.

## Evidence

**Established static evidence:** the supplied representation identifies incoming values, mixed-width accesses, ordered repetitions, saved-value uses, arithmetic expressions, and a final unresolved indirect transfer.

It distinguishes the word read at `0xA9` from the explicit byte writes at `0x3A`, `0x67`, and `0x21`. That distinction supports an absence-of-explicit-stores claim for the fixed-address read region, not unconditional preservation of its contents.

**Inferred intent:** no application-level interpretation is required by this contract.

**Unknowns:** target execution details, memory side effects, and dependency behavior remain unresolved as listed above.

No implementation access, independent instruction inspection, execution, or runtime tests have been performed. The acceptance examples are static deductions under their stated assumptions, not observed outcomes.
