<!-- Trusted provenance: F-b09637dbcccb; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-b09637dbcccb

## Purpose

F-b09637dbcccb conditionally transfers two consecutive 32-bit words from an indirectly addressed buffer into caller-provided storage, performs two all-one state writes, and invokes F-ddf4a1f94e38 twice.

A byte obtained through a shared global address determines whether the active path executes. A zero byte produces a false return without accessing either caller-provided region. A nonzero byte selects the active path and produces a true return if all subsequent operations complete normally.

The function reads the global address separately when saving a metadata base, locating the byte to test, and locating the byte to clear. These reads must not be collapsed into a single stable shared-state address. The tested byte and cleared byte belong to the same block only if the corresponding global reads yield the same address.

The apparent purpose is to consume a pending two-word result and invalidate associated state. That interpretation is inferred; the meanings of the flag, metadata, all-one values, and dependency arguments are not established.

## Interface

### Arguments and result

| Input | Established use |
|---|---|
| Object address | Represented as a signed 32-bit integer used as an address. On the active path, the function writes a 32-bit all-one value at displacement `0x80`, then passes the original argument to both dependency calls. |
| Output address | Identifies storage for two 32-bit words. The first destination is at byte displacement `0`; the second is at displacement `4`. |

The return is Boolean. It records whether the tested byte was nonzero. It does not report the values transferred, dependency success, or final memory contents.

The output words are raw 32-bit quantities. No interpretation as signed numbers, pointers, lengths, or domain-specific values is established.

### Global address and partial layouts

The function consults one global address location in three distinct roles:

| Role | When obtained | Uses |
|---|---|---|
| Saved metadata base | First global read, before the flag test | Buffer-address read, position-field read, and later metadata sentinel write |
| Tested flag base | Separate global read after saving the metadata base | Read the byte at displacement `0x02` |
| Cleared flag base | Separate global read, only after a nonzero test | Write zero to the byte at displacement `0x02` |

Only the first address is retained for the metadata operations. The cleared flag base is not obtained by reusing either earlier address.

The accessed layouts are partial:

| Base | Displacement | Width | Access |
|---|---:|---:|---|
| Tested flag base | `0x02` | 8 bits | Read and compare with zero |
| Cleared flag base | `0x02` | 8 bits | Write zero on the active path |
| Saved metadata base | `0x10` | 32 bits | Read as unsigned position metadata; later write `0xffffffff` |
| Saved metadata base | `0x14` | 32 bits | Read a signed integer used as the buffer address |
| Object address | `0x80` | 32 bits | Write `0xffffffff` |
| Output address | `0`, `4` | 32 bits each | Write the selected words |

No complete object size or shared-state structure is established.

### Preconditions for ordinary completion

The global address location must support the explicit reads. The tested flag location must be readable.

On the inactive path, neither caller argument is dereferenced by this function, and the saved metadata base is not used to access metadata.

On the active path, ordinary completion additionally requires:

- A writable flag byte through the separately obtained cleared flag base.
- Readable metadata at displacements `0x14` and `0x10` from the saved metadata base.
- A writable 32-bit field at displacement `0x10` from that same saved base.
- Readable source words at the calculated addresses.
- Writable storage for both output words and the object field.
- Address conversions and arithmetic that produce usable accesses in the execution environment.
- Both dependency calls to return normally.

The function contains no explicit validation of these requirements.

## Behavior

### Global-address reads and branch selection

The observable access sequence begins by reading the global address and saving its value as the metadata base.

The function then reads the global address again. Through this second value, it reads the byte at displacement `0x02` and records whether the byte is nonzero.

If the byte is zero:

- The third global-address read does not occur.
- No flag byte is cleared.
- No metadata field is read through the saved base.
- Neither source word is read.
- Neither caller-provided region is accessed.
- Neither sentinel write occurs.
- F-ddf4a1f94e38 is not called.
- The function returns false.

The first global read still occurs on this path, even though its saved value is not subsequently used for a memory access.

### Active-path access order

For a nonzero tested byte, the explicit sequence is:

1. Read the global address a third time.
2. Write zero to the byte at displacement `0x02` from that third address.
3. Read the buffer-address value at displacement `0x14` from the initially saved metadata base.
4. Read the unsigned position value at displacement `0x10` from that saved base and calculate the source index.
5. Read the first selected 32-bit source word and write it to output displacement `0`.
6. Read the next selected 32-bit source word and write it to output displacement `4`.
7. Write `0xffffffff` at displacement `0x10` from the saved metadata base.
8. Write `0xffffffff` at displacement `0x80` from the original object argument.
9. Call F-ddf4a1f94e38 with the original object argument and `0x80`.
10. After that call returns, call F-ddf4a1f94e38 with the original object argument and `0`.
11. Return true after the second call returns.

Thus, the flag clear precedes metadata reads and both transfers. It does not necessarily clear the byte that selected the active path.

If the global address changes between its first, second, and third reads, metadata access, flag testing, and flag clearing can involve three different blocks. A clear or output write that changes the global address does not redirect subsequent metadata accesses: those use the initially saved base, and there is no further explicit read of the global address.

The Boolean decision is retained. Neither the tested byte nor the cleared byte is reread to determine the result.

### Unsigned position calculation

The position field is read as an unsigned 32-bit value. Its calculation has two established unsigned stages:

- Logical right shift by two bits, discarding the lowest two bits.
- Subtraction of two using unsigned 32-bit arithmetic.

For shifted values of at least two, the unsigned result is the shifted value minus two. Otherwise, the subtraction wraps within 32 bits:

| Position value | Shifted value | Unsigned subtraction result |
|---|---:|---:|
| `0`–`3` | `0` | `0xfffffffe` |
| `4`–`7` | `1` | `0xffffffff` |
| `8`–`11` | `2` | `0` |
| `12`–`15` | `3` | `1` |
| `16`–`19` | `4` | `2` |

This unsigned result is then assigned to a signed integer. The multiplication by four and subsequent additions to the signed buffer-address value are expressed using signed integers.

For unsigned results within the signed 32-bit nonnegative range, the index value is representable. For `0xfffffffe` and `0xffffffff`, the conversion into the signed integer is not established by the supplied representation.

Even a representable index can produce an unrepresentable signed product or address sum. For example, position `0x80000008` produces unsigned index `0x20000000`, whose product with four is `0x80000000`, outside the signed 32-bit positive range.

Accordingly, the contract does not universally define effective addresses using modulo-\(2^{32}\) arithmetic. Unsigned subtraction wrap is established; signed conversion and signed-overflow outcomes require additional execution-model evidence.

Under a model that converts the two underflow results to signed values of minus two and minus one, their displacements are respectively minus eight and minus four bytes. This describes accesses before the buffer only if the subsequent address expressions also have the stated arithmetic meaning. It is not an unconditional result.

For representable, non-overflowing calculations, index zero selects the buffer address and the following word; index one selects displacements four and eight; index two selects displacements eight and twelve. The second source expression adds a further four bytes.

No lower-bound, upper-bound, or low-bit-alignment check is present.

### Transfer ordering and aliasing

The operation is two sequential word transfers, not an atomic eight-byte snapshot. The first output write precedes the second source read.

If the first output word occupies the second source word’s location, the first write can change the value read for the second transfer. Other overlaps can also affect behavior:

- The flag clear can alter metadata, a source word, output storage, object storage, or the global address if those locations overlap.
- Output writes can overwrite metadata, the global address, or object storage.
- The metadata sentinel can overwrite an output word.
- The later object sentinel can overwrite output or metadata storage.

The buffer-address value and calculated index have already been obtained before either output write. Changes to their original storage do not trigger a reload for the second transfer.

No overlap check or rollback is present. An interruption after the flag clear can leave partial output and partial state updates.

### Dependency completion

The second dependency call occurs only after the first returns. No dependency result is inspected, and no visible retry or recovery path exists.

The function performs no restoration after either call. A true return therefore does not guarantee that any flag is zero, either sentinel remains present, or the output still contains the transferred words. Those final-state claims require additional restrictions on aliasing, external activity, and dependency effects.

## Side effects

On the active path, the function explicitly performs:

- Three total reads of the global address, including the two preceding branch selection.
- One flag-byte read and one flag-byte clear, potentially in different blocks.
- Two metadata reads through the saved base.
- Two source-word reads and two output-word writes.
- One metadata sentinel write and one object sentinel write.
- Two ordered calls to F-ddf4a1f94e38.

On the inactive path, the explicit memory accesses are the first two global-address reads and the flag-byte read.

The sentinel is the 32-bit all-one pattern, `0xffffffff`. Its signed interpretation would be minus one, but its semantic role is unknown.

No allocation or release appears directly in this function. Output storage must already exist. This does not establish ownership transfer, retained references, or dependency memory-lifetime behavior.

No explicit synchronization, transactional guarantee, bounds enforcement, or error-reporting operation is visible.

## Dependencies

### F-ddf4a1f94e38

| Call | First argument | Second argument | Placement |
|---|---|---|---|
| First | Original object argument | `0x80` | After both transfers and both sentinel writes |
| Second | Original object argument | `0` | After the first call returns |

Established facts are limited to these arguments, ordering, and absence of result inspection.

The dependency’s purpose, return contract, memory effects, ownership effects, global-state effects, and abnormal-completion behavior are unknown. Neither argument value establishes that the second argument denotes an offset, command, or mode.

The second call receives the original object argument even if the first call changes global state or invalidates associated storage. The function contains no intervening validity check.

## Acceptance examples

These are conditional criteria derived from static evidence, not executed tests.

### Inactive state with different saved and tested bases

Suppose the first global read supplies one address, while the second supplies another address whose byte at `0x02` is zero.

The function returns false without accessing metadata through the first address. It does not perform a third global read, clear a flag, access either caller-provided region, or invoke the dependency.

### Stable-global ordinary extraction

Suppose all three global reads supply the same valid base, its tested byte is nonzero, its position field is `16`, and all address calculations are representable. Assume distinct storage regions and normal dependency returns.

The function clears that block’s flag, transfers buffer words at byte displacements eight and twelve, writes both sentinels, and makes the calls with second arguments `0x80` and `0`. It then returns true.

### Different test, clear, and metadata blocks

Suppose the three global reads supply three different valid bases. The second base’s flag is nonzero, the third base’s flag is writable, and the first base contains valid metadata.

The function clears the third base’s flag but obtains the source metadata and writes the metadata sentinel through the first base. It performs no explicit clear of the second base’s flag unless that byte aliases another written location.

### Low-bit equivalence

With otherwise equivalent metadata, valid storage, and representable address arithmetic, positions `16`, `17`, `18`, and `19` all produce index two and select the same source locations.

Their low bits are discarded rather than rejected.

### Conditional underflow interpretation

For position `4`, unsigned subtraction produces `0xffffffff`.

If the execution model converts that value to signed minus one, and the subsequent signed expressions have the expected representable results, the first source is four bytes before the buffer and the second is at the buffer address.

Without those arithmetic assumptions, the effective addresses remain documented_only. In either case, the flag clear precedes source-address use.

### Signed multiplication boundary

For position `0x80000008`, the unsigned calculation produces `0x20000000`.

That index is representable as a signed 32-bit value, but multiplying it by four is not. The supplied representation does not establish a successful access at a wrapped address, an explicit rejection, or a particular failure outcome.

### Overlapping transfer

Suppose the first output word occupies the second source word’s location, with no other relevant aliasing or intervening changes. Let the original source words contain distinct values A and B.

The first transfer replaces B with A. The second source read then obtains A, so the two transfers do not preserve the original pair.

### Dependency mutation

If the first dependency changes the saved block’s metadata sentinel and the second leaves it unchanged, the function does not restore the sentinel. It still returns true if both calls return normally.

## Uncertainties

**Inferred intent:** A pending-result flag, byte-count metadata, trailing-word extraction, and state invalidation are plausible interpretations, not established semantics.

**documented_only — Global stability:** No guarantee makes the separate global reads equal. Concurrency, reentry, and access-sensitive storage behavior are unspecified.

**documented_only — Integer and address model:** Unsigned shift and subtraction are established. Out-of-range unsigned-to-signed conversion, signed overflow, integer-to-address interpretation, and effective addresses in those cases are unresolved.

**documented_only — Representation:** Byte order, alignment requirements, memory-map constraints, and invalid-access behavior are not established.

**documented_only — Ownership:** Allocation provenance, object lifetime, output lifetime, and dependency retention or release behavior are unknown.

**documented_only — Errors:** The Boolean is not an error-status channel. Invalid accesses and abnormal dependency completion have no specified recovery behavior.

**documented_only — Final state:** Final memory contents require dependency contracts and assumptions about aliasing and external mutation that are not supplied.

## Evidence

Static evidence establishes three separately expressed global-address reads on the active path, retention of the first address for metadata operations, separate flag-test and flag-clear address acquisition, unsigned position reduction followed by signed expressions, ordered word transfers, sentinel writes, dependency arguments, and retained Boolean selection.

The supplied representation does not establish a universal wrapping address model or equality among the three global-address values. No dependency implementation, caller contract, memory map, or runtime observations were supplied.

No runtime tests have been performed.
