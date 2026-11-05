<!-- Trusted provenance: F-9addaaed381d; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-9addaaed381d

## Purpose

F-9addaaed381d initializes selected fields of a destination object, obtains an address from a dependency, and uses that address and a dependent construction result to initialize additional storage. On ordinary completion, it returns the original destination address.

The established contract concerns field widths, access order, dependency calls, and a zero/nonzero branch. It does not establish a general copy operation: only one source word and one source byte are copied directly. A further source word is passed to a dependency.

The resulting fields resemble a header-based linked structure, with self-referential fields when the construction result is zero. “Header,” as used below, is a descriptive label for the addressed storage, not proof of a particular container type, allocation policy, or ownership model.

## Interface

### Inputs and return value

The function accepts:

| Input | Representation | Established use |
|---|---|---|
| Destination address | Address in the supplied 32-bit representation | Receives direct initialization writes and is passed to two dependencies |
| Source address | Address in the supplied 32-bit representation | Supplies metadata and an address used for a nested read |
| Supplied byte | 8 bits | Written to destination offset `0x18` |

On ordinary completion, the returned value is the destination address supplied by the caller. No separate status value is returned.

All offsets below are byte offsets. Words and address-bearing fields occupy 32 bits. These widths describe the supplied representation; they do not establish compatibility with a wider-pointer environment.

### Destination fields directly accessed

| Offset | Width | Direct action |
|---|---:|---|
| `0x00` | 32 bits | Initially cleared |
| `0x04` | 32 bits | Initially cleared |
| `0x08` | 32 bits | Initially cleared |
| `0x0C` | 32 bits | Initially cleared |
| `0x10` | 32 bits | Initially cleared, then assigned the result of F-a81a4de4b94b; subsequently reread |
| `0x14` | 32 bits | Assigned the word read from source offset `0x14` |
| `0x18` | 8 bits | Assigned the supplied byte |
| `0x19` | 8 bits | Assigned the byte read from source offset `0x19` |

These are access descriptions, not unconditional final-value guarantees. Dependencies and overlapping header storage can subsequently modify the same bytes.

There is no direct initialization of destination offsets `0x1A` or `0x1B`.

### Source fields directly accessed

The function reads:

- A 32-bit word at source offset `0x14`.
- An 8-bit byte at source offset `0x19`.
- A 32-bit address-bearing word at source offset `0x10`, followed by a 32-bit read at offset `0x04` through that address.

The source reads do not form an initial snapshot. Each occurs at the point specified under Behavior.

### Header fields directly accessed

The result of F-a81a4de4b94b is stored in destination offset `0x10`. The returned address itself receives an immediate byte write. Later header accesses use addresses reread from destination offset `0x10`.

| Header offset | Width | Direct action |
|---|---:|---|
| `0x00` | 8 bits | Cleared through the address returned by F-a81a4de4b94b |
| `0x04` | 32 bits | Receives the construction result; reread on the nonzero branch |
| `0x08` | 32 bits | Receives a header address or the result of F-c5e056907a6d |
| `0x0C` | 32 bits | Receives a header address or the result of F-da8d15161b11 |

The byte write at `0x00` does not clear header bytes `0x01` through `0x03`.

### Preconditions for ordinary completion

Every address actually used must support the required read or write at that time, including any alignment requirements of the execution environment. Fixed-offset address calculations must reach the intended storage.

In particular:

- Destination storage must support the listed direct accesses.
- Source storage must support its reads when they occur.
- The address read from source offset `0x10` must support the nested word read at offset `0x04`.
- The address returned by F-a81a4de4b94b must support the immediate byte write.
- Each address subsequently read from destination offset `0x10` must support the accesses performed through it.

There is no established non-overlap requirement enforced by the function. Overlap instead affects the results and may affect whether subsequent accesses remain valid.

Initial argument validity alone is insufficient to guarantee completion. Dependencies can change storage that this function later rereads.

## Behavior

### Initial destination writes

Before calling any dependency, the function performs the following sequence:

1. Clears the destination word at `0x00`.
2. Clears the destination word at `0x10`.
3. Reads the source word at `0x14` and writes it to destination `0x14`.
4. Writes the supplied byte to destination `0x18`.
5. Reads the source byte at `0x19` and writes it to destination `0x19`.
6. Clears the destination word at `0x0C`.
7. Clears the destination word at `0x08`.
8. Clears the destination word at `0x04`.

The ordering matters for overlapping storage. For example, a destination write may change a source location that has not yet been read.

These accesses establish sequential effects in the supplied representation. They do not establish atomic initialization, thread safety, or permission for unsynchronized concurrent access.

### Header acquisition

The function calls F-a81a4de4b94b once, passing the destination address.

After that call returns, two direct operations occur in order:

1. The returned address is stored in destination offset `0x10`.
2. A zero byte is written through the returned address at offset `0x00`.

The second operation uses the returned address itself. It is not specified as a fresh read of destination offset `0x10`.

There is no null check, validity check, or explicit error-status branch between the call and these writes. If the returned address overlaps destination or source storage, the byte write can modify that storage before the construction arguments are read.

### Construction call

The function calls F-a896d89b4b1f with three arguments:

1. The destination address.
2. The 32-bit word obtained by reading the address at source offset `0x10`, then reading a word at offset `0x04` through that address.
3. The current 32-bit value at destination offset `0x10`.

The memory reads forming the second and third arguments occur after header acquisition, the destination header-field assignment, and the zero-byte write. They therefore observe any relevant changes caused by those operations or by F-a81a4de4b94b.

The supplied representation does not establish a relative evaluation order between the second argument’s memory reads and the third argument’s read. Neither argument should be treated as an earlier snapshot.

After F-a896d89b4b1f returns, the function rereads destination offset `0x10` and stores the returned 32-bit value at offset `0x04` of that header.

The branch then tests the returned value itself for zero. It does not reread header offset `0x04` to select the branch. Consequently, even if storing the result changes other relevant storage through aliasing, branch selection still depends on the dependency’s returned value.

### Zero-result branch

When F-a896d89b4b1f returns zero:

- A header address read from destination offset `0x10` is written into offset `0x08` of the header designated by that field.
- Destination offset `0x10` is read again after this write.
- This last-read address is both the value written and the base address for the final word write at offset `0x0C`.

The expression producing the offset-`0x08` write does not establish a relative evaluation order between its address-bearing reads. Under stable ordinary memory, both identify the same current header.

Neither F-c5e056907a6d nor F-da8d15161b11 is called on this branch.

With stable, non-overlapping header storage, the resulting header fields are:

- `0x04`: zero.
- `0x08`: the header’s own address.
- `0x0C`: the header’s own address.

These are established field effects under the stated conditions. Interpreting them as an empty container or sentinel state is inferred intent.

### Nonzero-result branch

When F-a896d89b4b1f returns any nonzero 32-bit value:

1. F-c5e056907a6d is called with no explicit arguments.
2. After it returns, destination offset `0x10` is reread, and the dependency’s result is written to offset `0x08` of that header.
3. Destination offset `0x10` is read again, and the word at offset `0x04` of the designated header is obtained.
4. F-da8d15161b11 is called with that word.
5. After this call returns, destination offset `0x10` is reread.
6. The result of F-da8d15161b11 is written to offset `0x0C` of this last-read header.

The argument to F-da8d15161b11 is a fresh memory read, not necessarily the original construction result. Changes made by F-c5e056907a6d, or by the intervening offset-`0x08` write when storage overlaps, can affect that argument.

Likewise, the final write is not required to target the header used before F-da8d15161b11. That dependency may replace destination offset `0x10`.

### Completion and arithmetic

After the final offset-`0x0C` write, the function returns the original destination address.

The direct arithmetic consists of fixed byte-offset address calculations. There is no direct loop, count multiplication, size calculation, or capacity check. No bounds or overflow checks are shown for the address calculations; the contract does not assume that wrapping an invalid address produces valid storage.

The construction result is tested only for its zero bit pattern. A nonzero value is not classified as positive, negative, valid, or erroneous. Results from the final two dependencies are stored without a separate zero/nonzero test.

## Side effects

### Memory and dependency effects

Direct writes affect the destination fields and addressed header fields described above. The zero-result path invokes two dependencies; the nonzero-result path invokes four.

Dependencies may have further effects. No dependency is assumed to preserve globals, source storage, destination storage, or previously obtained headers.

The function does not directly write through the source address as such. Nevertheless, source storage can change through aliasing with destination or header writes, or through dependency effects.

### Aliasing

No overlap detection or special self-initialization path is present.

If source and destination are identical, the original word at offset `0x10` is cleared and later replaced before the nested source-header read occurs. That read therefore does not reliably use the original source header.

Header overlap is independently significant. Disjoint source and destination objects do not imply that the address returned by F-a81a4de4b94b is separate from either object.

For example, if that dependency returns the destination address and later dependencies leave the relevant storage unchanged, zero-result header writes at offsets `0x08` and `0x0C` overwrite the corresponding destination words with the destination address. Their initial clearing does not make their final values zero.

No general alias-safe copy guarantee is established.

### Ownership and lifetime

The function stores an acquired address in the destination, but the supplied body does not establish:

- Whether the address denotes newly allocated, reused, borrowed, or shared storage.
- Whether ownership transfers to the destination or caller.
- Whether dependencies retain references.
- Who must release the header or any associated objects.
- Whether dependencies release earlier resources.

No direct release operation appears. Existing destination contents are overwritten without an explicit disposal operation in this body. This does not establish that dependencies perform disposal or that overwriting an already-initialized object is supported.

### Errors and partial state

There is no explicit rollback, exception handling, null-result recovery, or error translation.

A zero construction result selects the zero-result branch; the body does not distinguish emptiness from failure encoded as zero.

If an access fails or a dependency does not return normally, earlier writes may remain visible. The function provides no established transactional guarantee and no defined error return for such cases.

## Dependencies

| Opaque ID | Invocation | Established use of result | Unresolved behavior |
|---|---|---|---|
| F-a81a4de4b94b | Once, with destination address | Stored at destination `0x10`; returned address receives a zero-byte write | Acquisition mechanism, validity, ownership, failure behavior, mutations |
| F-a896d89b4b1f | Once, with destination, nested source word, and current destination header value | Stored at current header `0x04`; returned value selects the branch | Argument semantics, construction or copying behavior, resource effects |
| F-c5e056907a6d | Once on the nonzero branch, with no explicit arguments | Stored at current header `0x08` | Hidden inputs, global dependence, mutations, result meaning |
| F-da8d15161b11 | Once on the nonzero branch, with freshly read header word at `0x04` | Stored at offset `0x0C` of the header reread after the call | Traversal or other semantics, mutations, ownership effects |

The lack of explicit arguments to F-c5e056907a6d does not establish purity or independence from preceding operations.

## Acceptance examples

These are conditional criteria derived from static evidence. They are not executed tests.

### Separate storage, stable header, zero construction result

Assume:

- The accessed source and destination storage are disjoint.
- The accessed header storage, including its byte at `0x00` and words at `0x04`, `0x08`, and `0x0C`, is disjoint from all accessed source and destination storage.
- All accesses are valid.
- Dependencies leave relevant memory unchanged except for effects explicitly performed by this function.
- F-a81a4de4b94b returns header address `H`.
- F-a896d89b4b1f returns zero.

Accept if:

- Destination words at `0x00`, `0x04`, `0x08`, and `0x0C` are zero.
- Destination `0x10` contains `H`.
- Destination `0x14` contains the source word read during initialization.
- Destination `0x18` contains the supplied byte.
- Destination `0x19` contains the source byte read during initialization.
- Header `H` has a zero byte at `0x00`, a zero word at `0x04`, and `H` at both `0x08` and `0x0C`.
- Neither nonzero-branch dependency is called.
- The returned address is the original destination address.

Header bytes `0x01` through `0x03` retain their previous values under these assumptions; clearing them is not required.

### Separate storage, stable header, nonzero construction result

Use the same validity, stability, and pairwise non-overlap assumptions as the preceding example. Let the construction result be nonzero `R`, F-c5e056907a6d return `L`, and F-da8d15161b11 return `T`.

Accept if:

- Header offsets `0x04`, `0x08`, and `0x0C` contain `R`, `L`, and `T`, respectively.
- F-da8d15161b11 receives `R`.
- Dependency order is F-a81a4de4b94b, F-a896d89b4b1f, F-c5e056907a6d, then F-da8d15161b11.
- The destination fields retain the initialization results described above.

These criteria do not apply unchanged when header storage overlaps either object.

### Header aliases the destination

Assume the source is disjoint, the destination address is nonzero, all accesses are valid, and dependencies otherwise leave relevant storage unchanged. Let F-a81a4de4b94b return the destination address and the construction result be zero.

Accept if destination offsets `0x08` and `0x0C` finish containing the destination address, not zero. The stable header address does not prevent header writes from overwriting destination fields.

### Mutation before the final dependency

Assume a nonzero construction result `R`. F-c5e056907a6d changes the word at offset `0x04` of the current header to `Q`, without replacing the header. Assume the subsequent offset-`0x08` write does not overlap that word.

Accept only if F-da8d15161b11 receives `Q`, rather than the earlier result `R`.

### Header replacement during the final dependency

Assume F-da8d15161b11 changes destination offset `0x10` to valid header address `J` and returns `T`.

Accept only if the final offset-`0x0C` write stores `T` into header `J`. A write to a header cached before the call would not preserve the established reread behavior.

### Null header result

If F-a81a4de4b94b returns zero, the next direct operation stores zero into destination offset `0x10`. Only afterward does the function attempt the zero-byte write through the returned zero address.

No graceful error return is required by the supplied body. The outcome of the attempted address-zero write depends on the execution environment. If execution stops there, the preceding destination write has already occurred.

## Uncertainties

The following remain **documented_only**:

- Semantic names and invariants for the object fields and header fields.
- Whether the structure is a tree, list, or another representation.
- Whether a zero construction result denotes emptiness, failure, or another state.
- Dependency contracts beyond their visible arguments and uses.
- Allocation, ownership, retention, and release obligations.
- Supported overlap arrangements and intended self-initialization behavior.
- Endianness, precise alignment rules, and invalid-access outcomes.
- Synchronization, concurrency, and access atomicity guarantees.
- Defined behavior for address calculations outside valid storage.
- Whether dependencies initialize fields not directly written here.

The sentinel-like interpretation is inferred intent only. It must not replace the explicit widths, offsets, ordering, and rereads.

## Evidence

The supplied static body establishes the direct accesses, their widths, the initialization sequence, dependency call order, branch condition, post-call rereads, and returned destination address.

The supplied dependency mapping establishes the opaque function IDs used in this contract. No dependency implementation establishes allocation behavior, semantic field roles, ownership, or preservation guarantees.

The conditional examples follow from those direct effects under their stated assumptions, including explicit header non-overlap where independent final destination and header values are required.

No runtime tests have been performed.
