<!-- Trusted provenance: F-ce3bd5ea2520; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-ce3bd5ea2520

## Purpose

F-ce3bd5ea2520 transforms one 16-byte input block into one 16-byte output block using caller-provided 32-bit schedule words and globally configured lookup tables.

The established operation consists of:

- Packing input bytes into four words in most-significant-byte-first order.
- Combining those words with an initial group of schedule words.
- Applying a count-controlled sequence of table-based transformations.
- Applying a final, byte-masked table transformation.
- Writing four result words as 16 bytes, again in most-significant-byte-first order.

The structure is consistent with a table-driven block-cipher operation. Identifying a particular cipher, direction, key size, or standard requires table contents and schedule-generation evidence that are not supplied. Those interpretations are not prerequisites for the behavioral contract below.

## Interface

The function has four positional inputs and no return value:

| Position | Meaning | Required representation |
|---|---|---|
| First | Schedule storage | Contiguous, readable 32-bit words in the target’s native word representation |
| Second | Transformation count control | Signed 32-bit integer |
| Third | Input block | At least 16 readable bytes |
| Fourth | Output block | At least 16 writable bytes |

No lengths, capacities, table descriptors, or error-output arguments are supplied.

### Widths and byte interpretation

Input and output elements are 8-bit bytes. State words, schedule words, table entries, masks, and exclusive-OR results are 32 bits.

Input bytes are treated as unsigned values. Each consecutive group of four input bytes becomes a word with the first byte in bits 31–24 and the fourth byte in bits 7–0. Output performs the inverse byte placement.

Schedule words and table entries are read as native 32-bit words, not assembled using this input-byte convention. Their byte representation in memory therefore depends on the target’s native word layout.

### Supported count domain

For the normally terminating behavior described here, let:

- \(r\) be the signed count argument.
- \(h=\lfloor r/2\rfloor\), for positive \(r\).

The supported domain is \(h\geq1\), equivalently positive counts of at least 2.

The operation performs:

- \(2h-1\) full table transformations;
- one final masked transformation;
- \(2h\) transformations in total after the initial schedule combination.

It requires \(4(2h+1)\) schedule words, or \(16(2h+1)\) bytes, assuming ordinary nonwrapping address arithmetic and sufficient accessible storage.

An odd positive count is effectively rounded down to the preceding even count. There is no explicit rejection of odd counts.

### Preconditions

A conforming ordinary-memory invocation requires:

- Valid input and output ranges of 16 bytes.
- Sufficient readable schedule storage for the effective count.
- Valid globally configured table regions.
- Appropriate alignment for native word reads where required by the target.
- No address overflow or inaccessible memory in any schedule or table access.
- A count in the supported domain.

Stable schedule and table contents are needed for the mathematical mapping in this specification to describe a single deterministic transformation. Aliasing and mutation can make the actual access sequence relevant, as detailed below.

## Behavior

### Lookup-table layout

One global address supplies the base of the fourth full-transformation table. Relative to that address, the four table bases are:

| Logical table | Relative byte displacement |
|---|---:|
| \(T_0\) | −3072 |
| \(T_1\) | −2048 |
| \(T_2\) | −1024 |
| \(T_3\) | 0 |

Each table is indexed by an unsigned byte and contains 256 addressable 32-bit entries. An index selects the word at four times that index from the corresponding base.

The accessed layout therefore spans four adjacent 1024-byte regions. This describes the required address layout; it does not establish allocation ownership or whether the regions belong to a single allocation.

A separate global address supplies a final table \(U\), also indexed as 256 32-bit entries.

No table contents are supplied. In particular, the four byte lanes of a final-table entry must not be assumed identical.

### Word and byte notation

For a 32-bit word \(w\), let \(b_0(w)\), \(b_1(w)\), \(b_2(w)\), and \(b_3(w)\) denote its bytes from most significant to least significant.

Let \(K_j\) denote schedule word \(j\), numbered from zero relative to the original schedule pointer.

All combinations below use 32-bit bitwise exclusive OR, denoted \(\oplus\). Table indices are always in the range 0–255.

### Initial state

The input is read as four consecutive four-byte groups. Each group is assembled in most-significant-byte-first order and combined with its corresponding initial schedule word:

\[
S_j=\operatorname{BE32}(\text{input bytes }4j\ldots4j+3)\oplus K_j,
\qquad j=0,1,2,3.
\]

The four initial state words are established before any output byte is written.

The visible initialization proceeds by input groups at offsets 0, 4, 8, and 12. It does not establish an independent ordering among the byte and schedule reads within one expression.

### Full transformations

For a state \(S\) and a group of four schedule words beginning at index \(q\), a full transformation produces state \(V\) according to:

\[
V_j=
T_0[b_0(S_j)]
\oplus T_1[b_1(S_{(j+1)\bmod4})]
\oplus T_2[b_2(S_{(j+2)\bmod4})]
\oplus T_3[b_3(S_{(j+3)\bmod4})]
\oplus K_{q+j}.
\]

Every result word depends on the incoming state, not on partially updated words from the same transformation.

The first transformation uses schedule words 4–7. Each subsequent full transformation uses the next four schedule words. Thus full-transformation number \(t\), numbered from 1, uses words \(4t\) through \(4t+3\).

The loop computes one full transformation before checking whether its half-count has reached zero. When more work remains, it computes a second full transformation and advances to the next pair.

Within the first transformation of each pair, result words are visibly evaluated in logical order 0, 1, 3, 2. Within the second transformation, they are evaluated in order 0, 1, 2, 3. This ordering does not affect the mathematical result for stable ordinary memory, but can matter if schedule or table reads have side effects.

The specification does not impose a left-to-right order on the individual table and schedule reads inside one exclusive-OR expression.

### Count progression and schedule consumption

The half-count is decremented once per loop iteration, after the first transformation in that iteration has begun and before the exit decision.

For supported counts:

- There are exactly \(h\) loop iterations.
- Every iteration performs its first transformation.
- Only the first \(h-1\) iterations perform their second transformation.
- The final masked transformation uses schedule words \(8h\) through \(8h+3\).

Every schedule group from the initial group through the final group is used; no schedule group is skipped.

The count controls traversal only. It is not included as data in the word transformations.

### Final masked transformation

Let \(S\) now denote the state after all full transformations. Define four lane masks:

\[
M_0=\text{0xFF000000},\quad
M_1=\text{0x00FF0000},\quad
M_2=\text{0x0000FF00},\quad
M_3=\text{0x000000FF}.
\]

For stable final-table configuration, output word \(Y_j\) is:

\[
\begin{aligned}
Y_j={}&
(U[b_0(S_j)]\mathbin{\&}M_0)\\
&\oplus(U[b_1(S_{(j+1)\bmod4})]\mathbin{\&}M_1)\\
&\oplus(U[b_2(S_{(j+2)\bmod4})]\mathbin{\&}M_2)\\
&\oplus(U[b_3(S_{(j+3)\bmod4})]\mathbin{\&}M_3)\\
&\oplus K_{8h+j}.
\end{aligned}
\]

Each lookup contributes only its designated byte lane. The other 24 bits of that entry do not affect that contribution.

The final transformation does not use the four full-transformation tables.

### Output serialization and ordering

Output words are computed and stored one at a time, in logical order 0, 1, 2, 3.

For each output word, four byte writes occur in order from most significant byte to least significant byte. Consequently, explicit output stores proceed through offsets 0–15 in increasing order.

The first four output bytes are written before the second output word is computed. Likewise, bytes 4–7 are written before the third word is computed, and bytes 8–11 before the fourth word is computed.

This is not an atomic 16-byte output operation. A failure or externally visible interruption during output can leave a partially written result.

### Invalid counts

There is no guard for a count whose shifted half-count is zero or negative.

In particular, counts 0 and 1 do not request an empty operation: the function enters the full-transformation loop, decrements zero to −1, and does not take the intended first exit.

Negative counts likewise lack a supported successful interpretation. Continued traversal may encounter inaccessible schedule memory, address overflow, or signed-arithmetic limits. Eventual wraparound termination is not a contract: the signed shift and overflow behavior needed to assert it are not established.

## Side effects

### Explicit memory effects

The function explicitly writes only the 16 output bytes. It does not explicitly modify the schedule, input bytes, or global address slots.

That statement is about write destinations, not physical noninterference. If output storage overlaps schedule storage, table storage, or global address storage, those underlying objects may be modified by output writes.

No allocation, deallocation, ownership transfer, or memory clearing is visible. The caller retains responsibility for all supplied storage.

### Overlap behavior

Input/output overlap is compatible with the observed access order when the schedule and tables remain unaffected: all input bytes are consumed before output writes begin. Exact in-place transformation and partial overlap with the input do not, by themselves, cause input bytes to be reread after overwrite.

Output overlap with the schedule is different. Final schedule words are read separately as each output word is computed. Earlier output writes can therefore overwrite schedule words needed for later output words.

Similarly, output overlap with the final table can change later lookups. The contract does not promise correct stable-table results for such overlap.

### Global address observations

The final-table address is captured near entry. The last three output words use that captured address. The first output word instead refers again to the global final-table address during its computation.

For the full transformations, the bases of the first three tables are derived during initialization, while fourth-table accesses refer to the global base expression.

Thus a snapshot of all table configuration for the entire call is not established. The distinction is immaterial for stable ordinary memory, but matters if global address storage changes or aliases the output. No synchronization or concurrent-mutation guarantee is supplied.

### Errors and partial effects

There is no status return, bounds check, null check, count validation, or explicit error path.

Invalid memory or unsupported arithmetic is not converted into a reported error. The precise fault mechanism and recovery behavior depend on the execution environment.

Before output begins, the visible work consists of reads and internal computation. During final processing, an error can occur after an earlier output prefix has already been stored.

Table addresses depend on input-derived state. Constant-time or cache-independent behavior is not established.

## Dependencies

No function calls are visible. There are no additional opaque function dependencies to identify.

The required data dependencies are:

- The caller’s schedule words.
- The caller’s 16-byte input.
- The caller’s 16-byte output destination.
- A global full-table base and its four associated lookup regions.
- A global final-table base and its lookup region.

Table initialization, schedule generation, global configuration, and allocation lifetimes are outside the supplied evidence. No guarantees about those activities are imported into this contract.

## Acceptance examples

These are conditional acceptance criteria derived from static evidence, not executed tests.

### Minimum supported count

Given count 2 and valid stable storage:

- Twelve schedule words are required.
- Words 0–3 initialize the state.
- Words 4–7 participate in one full transformation.
- Words 8–11 participate in the final masked transformation.
- Exactly 16 output bytes are written.

Count 3 has the same transformation and schedule requirements as count 2.

### Larger count

Given count 10 and valid stable storage:

- Five loop iterations occur.
- Nine full transformations precede the final transformation.
- Forty-four schedule words are required.
- Final schedule words are indices 40–43.

Count 11 produces the same result as count 10 when all other inputs and relevant memory contents are identical.

### Byte-order check

If the first four input bytes are 0x12, 0x34, 0x56, and 0x78, the first packed input word is 0x12345678 before its initial schedule combination.

If a computed final word is 0x89ABCDEF, its output bytes are 0x89, 0xAB, 0xCD, and 0xEF, in that order.

These byte-order rules do not determine the native in-memory representation of schedule or table words.

### Synthetic zero-table check

Given count 2, full tables containing only zero words, a final table containing only zero words, and a disjoint output:

- Output depends only on schedule words 8–11.
- Those four words appear in most-significant-byte-first byte order.

This follows because all table contributions are zero, including those in the final transformation. It does not describe the unknown production tables.

### In-place check

Given supported count, valid stable schedule and tables, and the same address for input and output, the result must equal the result obtained with disjoint input and output buffers initialized with the same input bytes.

This criterion excludes overlap with schedule, tables, or global configuration storage.

### Unsupported-count boundary

For count 0 or 1, acceptance must not require a no-op, unchanged output, successful return, or reported validation error. None of those behaviors is established.

## Uncertainties

The following remain **documented_only**:

- Actual lookup-table contents and their provenance.
- Cipher identity, transformation direction, and cryptographic guarantees.
- How the schedule is generated and which counts its producer permits.
- Target-native endianness and alignment requirements for word storage.
- Negative signed-shift behavior and behavior after signed overflow.
- Exact machine-level ordering within individual read expressions.
- Memory-mapped-I/O behavior, concurrency semantics, and fault handling.
- Whether surrounding callers impose stricter nonoverlap or count constraints.

The cipher-like interpretation is inferred intent. The byte layout, table-index relationships, count progression for positive supported counts, schedule consumption, and output-store order are established by the supplied static evidence.

## Evidence

This contract is based only on the supplied body of F-ce3bd5ea2520.

The static evidence establishes four-byte input packing, 32-bit schedule and table operations, byte-indexed lookup addresses, a paired transformation loop, a distinct masked final transformation, and sixteen ordered output-byte stores.

It also establishes the absence of visible calls, explicit allocation, validation branches, and error reporting within this function.

No table definitions, caller contracts, schedule producer, runtime observations, or executable environment were supplied. No runtime tests have been performed.
