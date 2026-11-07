<!-- Trusted provenance: F-3eaa60da10c7; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-3eaa60da10c7

## Purpose

F-3eaa60da10c7 imports a sequence of bytes into a word-oriented destination object. For a nonnegative byte count, the visible behavior is consistent with interpreting the input as an unsigned, big-endian integer and storing its magnitude in 32-bit words, least-significant word first.

The function updates the destination’s significant-word count and removes zero-valued most-significant words from that count. For a nonempty input, it also clears a separate metadata word.

The integer interpretation is inferred intent. The established behavior is the sequence of object-field updates, byte reads, 32-bit word writes, and final zero-word trimming described below. The meanings of the metadata fields and the dependency contracts are not independently established.

## Interface

### Inputs

The function accepts three arguments:

1. **Source address:** the beginning of a byte sequence.
2. **Byte count:** a signed 32-bit value.
3. **Destination address:** either an existing destination object or null.

The source is read, not explicitly written. A null source can be used on the zero-count path because that path performs no source-byte reads. A positive count requires enough readable source bytes to complete the conversion.

A nonnull destination must provide writable metadata and, when required, writable backing storage. There are no visible checks for invalid addresses, insufficient allocations, overlapping storage, or inconsistent metadata.

### Destination layout

The visible accesses identify four consecutive 32-bit fields:

| Byte offset | Established use |
|---|---|
| 0 | Address used as the base of the backing-word storage |
| 4 | Significant-word count |
| 8 | Capacity-like value used in a signed comparison |
| 12 | Metadata explicitly set to zero for nonempty input |

Backing-storage entries are 32-bit words at four-byte intervals. Word index zero receives the least-significant portion of the imported value.

Calling the field at offset 8 a capacity and the field at offset 12 a sign or status field is an interpretation, not an independently verified structure definition.

The displayed representation uses a 32-bit word to hold the backing-storage address. This contract follows that apparent 32-bit address model. A wider-pointer ABI is not established.

### Result

The result is either:

- null, following one of the two explicit dependency-failure branches; or
- the selected destination address.

The selected destination is the caller-supplied object when nonnull. Otherwise, it is the nonnull object returned by F-7d962cda8fd4.

A nonnull result from F-08bc2fc94856 is used only as a success indication. It does **not** replace the selected destination address used by subsequent accesses or returned to the caller.

### Preconditions for ordinary import behavior

The ordinary import interpretation requires:

- a nonnegative byte count;
- valid destination metadata and backing storage;
- sufficient readable source storage for a positive count;
- no harmful overlap between source bytes, destination metadata, and backing words;
- arithmetic and address calculations that remain meaningful in the target execution model;
- dependency behavior compatible with continued use of the selected destination.

The function does not enforce these conditions.

## Behavior

### 1. Select the destination

If the supplied destination is nonnull, the function uses it directly.

If it is null, the function calls F-7d962cda8fd4 with no explicit arguments. If that call returns null, the function immediately returns null. No source-byte reads or explicit destination-field writes occur before this return.

If the call returns nonnull, that address becomes the selected destination.

### 2. Handle a zero byte count

For a byte count of zero, the function writes zero to the significant-word count at offset 4 and returns the selected destination.

This path does not explicitly:

- inspect or change the backing-storage address;
- inspect or change the capacity-like field;
- clear the metadata field at offset 12;
- read the source;
- clear backing words.

Thus, a zero-length conversion and conversion of a nonempty all-zero sequence both produce a zero significant-word count, but they need not produce identical metadata or backing-storage contents.

### 3. Check the capacity-like field for nonzero counts

For a nonzero count, the function computes a threshold from the count, then compares the field at offset 8 against that threshold using signed interpretation.

For nonnegative counts whose intermediate signed arithmetic does not overflow, the threshold is:

**T = floor((8 × count + 47) / 32).**

The threshold calculation contains a sign adjustment before an arithmetic right shift. For a representable negative intermediate, that adjustment implements division by 32 with truncation toward zero, rather than rounding toward negative infinity.

If the signed capacity-like value is less than T, the function calls F-08bc2fc94856 with:

- the selected destination address; and
- a second, separately calculated integer.

For ordinary nonnegative, nonoverflowing counts, that second integer is:

**G = floor((8 × (count + 2)) / 32) + 1.**

T and G are not interchangeable, and neither is always equal to the number of imported words. Examples include:

| Byte count | Imported words before trimming | T | G |
|---:|---:|---:|---:|
| 1 | 1 | 1 | 1 |
| 2 | 1 | 1 | 2 |
| 3 | 1 | 2 | 2 |
| 4 | 1 | 2 | 2 |
| 6 | 2 | 2 | 3 |

If F-08bc2fc94856 returns null, the function immediately returns null. Its own conversion-specific writes to the word count, metadata, and backing words have not yet occurred. The dependency may nevertheless have modified state.

If the dependency returns nonnull, processing continues through the original selected destination. Returning a different object address is not sufficient to redirect the conversion.

When the signed capacity comparison succeeds without calling the dependency, conversion proceeds directly.

### 4. Initialize conversion metadata

After the capacity stage succeeds, the function calculates the initial word count using unsigned 32-bit arithmetic:

**W = floor(((count − 1) modulo 2³²) / 4) + 1.**

For positive counts, this is exactly the count divided by four, rounded upward.

The function then performs these writes in order:

1. W to the significant-word-count field.
2. Zero to the metadata field at offset 12.

These writes occur before the first source-byte read. Consequently, overlapping source and metadata can change the bytes later consumed.

### 5. Consume bytes and store words

For a positive count, source bytes are read once each, in increasing address order.

A 32-bit accumulator starts at zero. Each consumed byte becomes its low eight bits after the previous accumulator value is shifted left by eight bits. The accumulator is written to backing storage at the end of each group, then reset to zero.

The grouping is:

- an initial group of one, two, three, or four bytes, chosen so that all subsequent groups have four bytes;
- zero or more complete four-byte groups.

The first group is stored at index W − 1. Later groups are stored at successively lower indices, ending at index zero.

For example, a five-byte sequence with byte values 01, 23, 45, 67, 89 produces:

- word index 1: 0x00000001;
- word index 0: 0x23456789.

This specifies numerical word values, not the physical byte order of each stored word. Physical byte order follows the target machine’s 32-bit store representation.

At each group boundary, the backing-storage address is read from the destination metadata **before** the source byte completing that group is read. The completed accumulator is then written through the saved address at that group’s word index. The backing address is read again at subsequent group boundaries, rather than cached for the whole conversion.

For valid, nonoverlapping storage, this distinction does not change the imported value. With overlap, earlier metadata or backing writes can affect later byte reads or address reads. There is no snapshot, staging buffer, or overlap protection.

No extra backing word is cleared beyond the words written by the conversion.

### 6. Trim zero-valued high words

After consuming the input, the function examines the significant-word count as a signed value. If it is positive, the function obtains the backing-storage address and identifies the position immediately beyond the counted words.

It then examines words from the highest counted index downward:

- A nonzero word ends trimming immediately, and the function returns the selected destination.
- A zero word causes the significant-word count to decrease by one.
- Trimming continues while the resulting count remains positive.

The words themselves are not rewritten during trimming.

For a positive input count and valid storage:

- leading zero input groups do not remain counted;
- an all-zero input ends with significant-word count zero;
- a nonzero result has a nonzero highest counted word.

Backing words excluded by trimming remain present in memory. They are outside the final logical count.

### 7. Return

The function returns the selected destination after conversion and trimming.

There is no additional visible success flag, byte count, error code, cleanup call, or ownership transfer operation.

## Side effects

### Direct memory effects

Depending on the path, the function can:

- write the significant-word count;
- clear the metadata field at offset 12;
- read the capacity-like field;
- repeatedly read the backing-storage address;
- read the requested source bytes;
- write imported 32-bit words;
- read high backing words during trimming.

On the normal nonempty path, the order is capacity check and any dependency call, initial word-count write, metadata clear, interleaved source reads and backing writes, then trimming reads and count reductions.

### Ownership and lifetime

The function contains no explicit release operation.

When the destination is created by F-7d962cda8fd4 and a later capacity-related call fails, no visible cleanup occurs here. Whether this leaves an allocation requiring release depends on the dependencies’ undocumented ownership behavior.

No promise is established that F-08bc2fc94856 preserves the original object, preserves its backing allocation, or releases anything. Successful execution requires the original selected address to remain usable because the function continues accessing it.

### Failure and atomicity

Only null returns from the two dependencies produce explicit null-result branches. Invalid pointers, bad metadata, arithmetic problems, and insufficient backing storage have no explicit recovery path.

The operation is not transactional. Dependency effects are not rolled back, and conversion writes occur incrementally.

No global-state preservation guarantee can be inferred for either dependency.

## Dependencies

### F-7d962cda8fd4

Called only when the supplied destination is null.

- Explicit arguments: none.
- Observed result use: null indicates failure; nonnull supplies the selected destination.
- Required compatibility: the returned address must support subsequent metadata accesses.
- Undocumented behavior: initialization, allocation strategy, ownership, global effects, and release requirements.

### F-08bc2fc94856

Called only when the signed capacity-like field is below the calculated threshold.

- Explicit arguments: selected destination and the calculated integer G.
- Observed result use: null indicates failure; any nonnull value allows conversion to continue.
- The returned address is not adopted.
- Required compatibility: the selected destination must remain usable and provide sufficient storage afterward.
- Undocumented behavior: parameter units, allocation or relocation policy, metadata changes, failure-state guarantees, ownership, and global effects.

Its placement suggests capacity preparation, but that purpose remains inferred.

## Acceptance examples

These examples are conditional static expectations, not executed tests. Unless stated otherwise, storage is valid and nonoverlapping, arithmetic does not overflow, and required dependency calls succeed compatibly.

### Empty input preserves unrelated fields

Given a valid existing destination and count zero:

- the result is that destination;
- the significant-word count becomes zero;
- the metadata field at offset 12 is not explicitly changed;
- backing storage is neither read nor cleared;
- the source need not be readable.

### Single-byte import

Given one byte with value A5:

- word index zero becomes 0x000000A5;
- the final significant-word count is one;
- the metadata field at offset 12 becomes zero.

### Full-word import

Given bytes 12, 34, 56, 78:

- word index zero becomes 0x12345678;
- the final count is one.

If the capacity-like field is one, the visible comparison nevertheless invokes F-08bc2fc94856 with second argument two.

### Partial high word

Given bytes 01, 23, 45, 67, 89:

- index one becomes 0x00000001;
- index zero becomes 0x23456789;
- the final count is two.

### High zero-word removal

Given bytes 00, 00, 00, 00, 01:

- index one is written as zero;
- index zero is written as one;
- trimming reduces the count from two to one.

### Nonempty zero value

Given any positive-length sequence containing only zero bytes:

- the conversion writes zero to every initially counted backing word;
- the metadata field at offset 12 is cleared;
- trimming reduces the final count to zero.

### Dependency failures

If destination creation returns null, the function returns null before accessing the source.

If capacity preparation returns null, the function returns null before its own conversion metadata writes. No unchanged-state guarantee is made for effects of that dependency.

### Different nonnull preparation result

If F-08bc2fc94856 returns an address different from the selected destination, subsequent conversion still targets the selected destination. Any acceptance criterion requiring automatic relocation to the returned address would contradict the visible behavior.

## Uncertainties

- **Negative counts — documented_only:** There is no rejection branch. The decrementing loop does not treat a negative count as empty input. Under a wrapping 32-bit machine interpretation, it can entail an enormous number of iterations. Safe or useful negative-count behavior is not established.
- **Signed overflow — documented_only:** Multiplication and addition in the threshold calculations can exceed signed 32-bit range. The displayed source does not establish portable behavior for those cases. The simplified formulas above apply only without overflow.
- **Address overflow — documented_only:** Backing-address arithmetic and source advancement are unchecked.
- **Exact object type — inferred intent:** Unsigned multiprecision-integer import is consistent with the storage and trimming behavior, but the full object definition is unavailable.
- **Capacity units — unknown:** The dependency argument and capacity-like field appear related to words, but their external contracts are not supplied.
- **Aliasing — documented_only:** Overlap is not prohibited by a visible check, but stable integer-import semantics cannot be promised when writes affect future reads.
- **Concurrency and reentrancy — unknown:** There is no visible synchronization, and dependency effects are unknown.
- **Fault behavior — unknown:** The function supplies no explicit handling for invalid memory accesses.

## Evidence

Established static evidence consists of the supplied function body and the opaque dependency mapping. It shows destination selection, explicit null-failure branches, field offsets, signed and unsigned arithmetic, byte-consumption order, backing-word writes, and high-zero-word trimming.

The unsigned-integer interpretation, capacity-preparation purpose, and possible meaning of the cleared metadata are inferences, distinguished above from direct observations.

Dependency implementations, an authoritative structure definition, and target ABI documentation were not supplied. No runtime tests have been performed.
