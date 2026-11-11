<!-- Trusted provenance: F-132a2eaccb34; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-132a2eaccb34

## Purpose

F-132a2eaccb34 computes a table-dependent, 32-bit checksum update over an input region. It accepts a prior checksum, an input address, and a byte count, and returns the updated checksum.

The visible behavior includes complementing the incoming checksum, processing input through byte-oriented and four-byte transformations, and complementing the final state. A null input address is handled separately: it returns zero regardless of the incoming checksum or byte count.

The structure is consistent with an optimized reflected CRC update. That interpretation is **inferred intent**, not an established checksum identity. The lookup-table contents, generating polynomial, target byte order, and known-answer vectors are not supplied. This contract therefore defines the observable table-dependent transformation without assigning a standard CRC name.

## Interface

### Inputs

| Input | Contract |
|---|---|
| Prior checksum | An unsigned 32-bit value, interpreted as an externally represented checksum rather than the complemented working state. |
| Input address | A nullable address of readable input storage. Although represented through a word-oriented pointer, progression and length are measured in bytes. |
| Byte count | An unsigned 32-bit count of logical input bytes to process. |

### Output

The result is an unsigned 32-bit value.

- A null input address produces zero.
- A non-null input address with zero count produces the prior checksum unchanged.
- Otherwise, the result is the complement of the final working state defined below.

There is no separate status value, output parameter, or visible error-reporting channel.

### Width and arithmetic model

The visible representation supports a 32-bit unsigned model: four-byte word spacing, four-byte table entries, four extracted eight-bit fields, and shifts up to 24 bits. This contract uses that model. Independent target ABI documentation is not present.

Under this model:

- Every checksum state and lookup-table entry is 32 bits.
- Complements invert exactly 32 bits.
- Right shifts of checksum values are logical, introducing zeros.
- Each extracted byte is in the range 0 through 255.
- XOR operations produce 32-bit results.
- Count reductions occur only after a corresponding size check; they do not underflow in the visible control flow.

Addresses must support the required accesses and progression without invalid address arithmetic. A large unsigned count is not interpreted as a negative length and receives no special rejection.

### Storage preconditions

For nonzero processing, the caller must provide valid readable storage for the accesses actually performed. Four-byte processing begins only at an address divisible by four.

There is an important qualification for byte-oriented processing: the supplied representation expresses a word dereference followed by narrowing to eight bits. It does not conclusively establish whether the actual memory access is one byte or four bytes. Consequently, a claim that exactly the requested byte range must be readable is **documented_only**, not established.

The lookup-table storage must also be readable. No validation of either input storage or table storage is visible.

## Behavior

### Table interpretation

The routine uses one externally supplied table base. For this contract, its accessible layout is described as four banks:

| Bank | Byte displacement from base | Entries |
|---|---:|---:|
| \(T_0\) | 0 | 256 unsigned 32-bit entries |
| \(T_1\) | 1024 | 256 unsigned 32-bit entries |
| \(T_2\) | 2048 | 256 unsigned 32-bit entries |
| \(T_3\) | 3072 | 256 unsigned 32-bit entries |

An entry with index \(i\) occupies the four-byte word beginning at its bank displacement plus \(4i\). The highest accessed entry begins at displacement 4092, so the described table spans 4096 bytes.

The contents of these banks are unknown. In particular, the supplied evidence does not establish that the banks are mutually consistent expansions of a single CRC table.

Define the following mathematical transformations to specify the result, not an implementation sequence.

For a 32-bit state \(s\) and an eight-bit input value \(b\), the byte transformation is:

\[
B(s,b)=T_0[(s\mathbin{\oplus}b)\mathbin{\&}255]
       \mathbin{\oplus}(s\gg8).
\]

For a 32-bit value \(x\), the four-byte table transformation is:

\[
Q(x)=
T_3[x\mathbin{\&}255]
\mathbin{\oplus}
T_2[(x\gg8)\mathbin{\&}255]
\mathbin{\oplus}
T_1[(x\gg16)\mathbin{\&}255]
\mathbin{\oplus}
T_0[x\gg24].
\]

These equations depend on the table values read during execution. Treating the tables as fixed mathematical arrays requires them to remain stable throughout the call.

### Null-address behavior

The input address is checked before checksum complementation, input processing, or table lookup.

If it is null:

- The result is zero.
- The prior checksum is ignored.
- The count is ignored, including a nonzero count.
- No input or table dereference is required by the visible path.

This is an explicit special case, not evidence of general argument validation.

### Initial and final representation

For a non-null input address, the working state begins as the 32-bit complement of the prior checksum. After all requested processing, the returned result is the 32-bit complement of that working state.

Thus, for a non-null address and zero count, the two complements cancel. No input or lookup-table read is needed on that path.

### Alignment prefix

Before using four-byte input words, the routine processes logical input bytes while both conditions hold:

- At least one byte remains.
- The current address is not divisible by four.

Each such step applies \(B\) to the working state and the eight-bit value selected by the byte-oriented input expression. The input address advances by one byte and the remaining count decreases by one.

At most three such steps occur before four-byte alignment is reached, unless the count is exhausted first.

The selected eight-bit value is represented as the low eight bits of a dereferenced word. Identifying that value with the byte located at the current address requires appropriate target byte order or confirmation of a narrow load. Neither is independently supplied.

### Bulk processing

After the alignment prefix, every available group of 32 bytes is processed as eight consecutive four-byte words.

If those words, in increasing address order, are \(w_0\) through \(w_7\), the resulting state is the composition:

\[
Q\bigl(w_7\mathbin{\oplus}
 Q\bigl(w_6\mathbin{\oplus}
 \cdots Q(w_0\mathbin{\oplus}s)\cdots\bigr)\bigr).
\]

This specifies eight successive state transformations, not eight independent checksums. The first word is combined with the incoming state; every later word is combined with the state produced by its predecessor.

After a complete group:

- The logical input position advances by 32 bytes.
- The remaining count decreases by 32.
- Processing repeats while at least 32 bytes remain.

The integer value of each four-byte input word follows the target’s word-load byte order. Little-endian interpretation is not independently established by the supplied material.

### Remaining whole words

After bulk processing, each remaining complete four-byte word updates the state to:

\[
Q(w\mathbin{\oplus}s).
\]

Each update advances the input position by four bytes and reduces the remaining count by four. This continues while at least four bytes remain.

### Final bytes

Any final one to three logical bytes are processed with the same byte transformation used for the alignment prefix. Each step advances the address by one byte and decreases the remaining count by one.

The final state is then complemented and returned.

The byte and word transformations are not proven equivalent for arbitrary supplied table contents. Consequently, alignment independence and equivalence between a bulk call and differently partitioned calls are conditional properties, not unconditional guarantees.

## Side effects

### Reads and ordering

The visible routine reads:

- Input values selected by the current processing path.
- The external table-base reference when lookup expressions require it.
- Table entries selected by evolving checksum state.

The processing phases are ordered: alignment prefix, 32-byte groups, remaining four-byte words, then final bytes. State updates within each phase depend on earlier updates.

Within an individual XOR expression, the supplied representation does not establish a strict evaluation order among independent operand reads. It therefore does not define a complete bus-access trace or an exact ordering between an input-word read and every associated table read.

No entry-time snapshot of the table base or table contents is established. The mathematical description assumes stable table storage. Behavior under concurrent mutation of that storage or its base reference is not specified as a deterministic checksum operation.

### Writes and ownership

No write to input storage, table storage, or other caller-visible memory is present in the visible body. Address and count progression are internal; the caller receives neither a modified pointer nor a remaining count.

No allocation, release, ownership transfer, or external function call is visible. The caller retains responsibility for input storage. The routine does not visibly initialize, repair, or replace the lookup tables.

### Errors

There is no visible bounds check, table-availability check, overflow error, exception translation, or recovery path.

Invalid non-null storage is outside the supported preconditions. The supplied evidence does not establish whether an invalid access traps, faults, or has another platform-specific effect. Returning zero for a null address must not be generalized to other invalid addresses.

## Dependencies

No callable dependency is visible in F-132a2eaccb34. No additional opaque function ID can therefore be assigned from the supplied dependency map.

Its external dependencies are data and machine semantics:

- A readable table-base reference.
- Four banks of 256 readable 32-bit entries.
- Target behavior for word loads, shifts, address arithmetic, and potentially unaligned byte-path accesses.
- Caller-provided readable input storage.

The table-generation mechanism, initialization timing, table ownership, and synchronization policy are unknown. No preservation or ownership guarantees are inferred from an unseen initializer or other unavailable function.

## Acceptance examples

These are conditional acceptance criteria derived from the supplied static representation. They are not executed tests.

1. **Null address takes precedence.**  
   With prior checksum `0x12345678`, null input, and count 100, the result is zero. The same result applies with count zero. Neither case requires reading the input or tables.

2. **Empty non-null input preserves the prior checksum.**  
   With a non-null address, count zero, and prior checksum `0x12345678`, the result is `0x12345678`. No input or table contents affect it.

3. **One byte uses only the first table bank.**  
   For count one, let \(b\) be the eight-bit value actually selected by the byte-oriented expression. The result is:
   \[
   \neg B(\neg c,b),
   \]
   where \(c\) is the prior checksum and both complements are 32-bit. This criterion does not resolve the physical load width.

4. **An aligned four-byte input uses all four banks.**  
   With an address divisible by four and count four, let \(w\) be the loaded 32-bit word. The result is:
   \[
   \neg Q(w\mathbin{\oplus}\neg c).
   \]
   No alignment-prefix or trailing-byte update occurs.

5. **Alignment can consume the entire request.**  
   If the address has remainder one modulo four and the count is two, both logical bytes use the byte transformation. No four-byte input word is processed.

6. **Alignment changes the bulk threshold reached.**  
   If the address has remainder three modulo four and the count is 33, one byte is processed first, followed by one eight-word group. There is no trailing remainder. Conversely, an aligned count of 31 uses seven whole-word updates and three final byte updates, with no 32-byte group.

7. **An aligned 36-byte request crosses both word-processing phases.**  
   It performs one eight-word group, followed by one remaining whole-word update. The final result is the complement of the ninth successive word-derived state.

8. **A partitioned update is conditionally equivalent.**  
   Feeding the first result back as the prior checksum cancels the intervening complement pair. Full equivalence with a single call additionally requires stable tables and consistency between byte and word transformations wherever partitioning changes their selection. Without that evidence, partition invariance is not an unconditional acceptance criterion.

## Uncertainties

The following remain **documented_only**:

- **Checksum identity:** no polynomial, standard designation, or numeric nonempty known-answer result is established.
- **Target byte order:** word loads are visible, but their byte interpretation is not independently documented.
- **Byte-path access width:** narrowing follows a represented word dereference; actual one-byte versus four-byte access is unresolved.
- **Alignment-independent results:** these require table and byte-order relationships not supplied.
- **Exact memory-access ordering:** state dependencies establish phase order, not a total order among independent expression operands.
- **Concurrent changes:** no locking, atomic snapshot, or mutation policy for the table base, table contents, or input is established.
- **Fault behavior:** invalid pointers, inaccessible tables, and address wraparound have no defined recovery contract here.
- **Completeness of the supplied representation:** annotations report omitted unreachable material, but that material is unavailable. It cannot justify additional behavior.

## Evidence

The established claims derive solely from inspection of the supplied function representation: the early null return, paired complements, alignment test, count thresholds, pointer increments, table offsets, shifts, masks, and absence of visible external calls or memory stores.

CRC intent is inferred from the complemented state convention and table-driven byte and word updates. It is not used to fill gaps in table contents, machine semantics, or standard checksum identity.

No runtime tests have been performed. The acceptance examples are static consequences or explicitly conditional requirements, not observed execution results.
