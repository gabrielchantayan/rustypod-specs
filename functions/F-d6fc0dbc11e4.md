<!-- Trusted provenance: F-d6fc0dbc11e4; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-d6fc0dbc11e4

## Purpose

F-d6fc0dbc11e4 performs an ordered sequence of initialization-like writes and dependency calls. It begins with caller-supplied storage, follows addresses returned by six dependencies, initializes fields relative to two of those results, and invokes two final dependencies with a derived address.

The established behavior is an ordered sequence of accesses and calls. It does not establish that all accesses belong to one contiguous object. Each of the first six dependency results determines the address used for the next step; no visible check requires a result to equal the address supplied to that dependency.

Constructor-like initialization is inferred intent. The object type, field meanings, ownership policy, and purposes of the final two operations remain unknown.

## Interface

### Inputs and result

| Input | Established representation | Direct use |
|---|---|---|
| First | Address of storage accessed initially as 32-bit words | Anchors the first two writes and the first dependency argument |
| Second | 32-bit value | Written at byte offset `+0x04` from the first input |
| Third | 32-bit value | Forwarded to F-226d97c76fa5 |
| Fourth | 32-bit value | Written at byte offset `+0x14` from the sixth dependency result |

The result is an integer-represented address equal to the sixth dependency result minus `0x54` bytes. The same derived address is supplied to both final dependencies.

The supplied representation supports 32-bit data fields and an apparent 32-bit integer/address model. It does not independently establish the original public declarations, calling convention, pointer provenance rules, or behavior of overflowing address arithmetic. The contract therefore does not prescribe wrapping or trapping for arithmetic outside the valid address range.

The first input and returned address are not necessarily equal. Their equality depends on the dependency results.

### Preconditions for normal completion

Normal completion requires:

- The first input to permit 32-bit writes at offsets `+0x00` and `+0x04`.
- The global source location to permit the initial 32-bit read.
- Every dependency argument to satisfy that dependency’s requirements.
- Addresses derived from the fourth and sixth results to permit the specified writes.
- All required alignment and address-arithmetic constraints to hold.
- Every dependency to return normally and leave subsequent accesses valid.

These are requirements for the described execution, not validations performed by this function. No storage size, explicit null check, returned-status check, or bounds check is visible.

## Behavior

### Global snapshot and initial writes

The function first reads one 32-bit value from a global location. This read precedes both initial destination writes.

It then performs, in order:

1. A 32-bit write of the second input at first-input offset `+0x04`.
2. A 32-bit write of the saved global value at first-input offset `+0x00`.

The first-word initialization uses the earlier snapshot. It does not reread the global after the second-input write.

This distinction matters when the destination aliases the global source. If the first write changes the global source location, the following write still uses the value read before that change.

The global value’s meaning is not established. Treating it as a dispatch pointer, descriptor, or type identifier would be an inference.

### Return-directed dependency chain

The first six calls occur in the following order. Every displacement in this contract is measured in bytes.

| Order | Dependency | Arguments |
|---|---|---|
| 1 | F-ad433e30ea59 | First-input address plus `0x08`; zero |
| 2 | F-fa70d5b4a668 | First dependency result plus `0x0c`; zero; zero |
| 3 | F-7caeaed12e0c | Second dependency result plus `0x10`; zero |
| 4 | F-95094c640cf9 | Third dependency result plus `0x0c`; zero |
| 5 | F-758c15808c75 | Fourth dependency result plus `0x10` |
| 6 | F-e511432e9f02 | Fifth dependency result plus `0x14` |

Between calls 4 and 5, the function writes a 32-bit zero at the fourth dependency result plus `0x0c`. Thus, that write occurs before F-758c15808c75 is invoked.

The chain follows returned addresses rather than repeatedly deriving locations from the first input. A redirected result changes the base for subsequent operations.

No dependency result is visibly classified as success, failure, null, or a status code. The zero-valued arguments are established only as numeric zeros; their meanings as flags, counts, modes, or null pointers are unknown.

### Writes relative to the sixth result

After F-e511432e9f02 returns, the following writes occur relative to its result:

| Write order | Byte offset | Width | Value |
|---|---:|---:|---|
| 1 | `+0x08` | 32 bits | Zero |
| 2 | `+0x0c` | 8 bits | Zero |
| 3 | `+0x0d` | 8 bits | Zero |
| 4 | `+0x0e` | 8 bits | Zero |
| 5 | `+0x0f` | 8 bits | Zero |
| 6 | `+0x14` | 32 bits | Fourth input |
| 7 | `+0x18` | 32 bits | `0xffffffff` |

There are **three 32-bit writes** and **four 8-bit writes** relative to this result.

The four byte writes are separate ordered operations. They are not specified as one 32-bit write, even though they leave four consecutive zero bytes when no other effects intervene.

After the fourth byte write and before the fourth-input write, the function calculates the eventual returned address by subtracting `0x54` from the sixth result. This calculation does not itself establish a memory read or write. Its position matters if invalid arithmetic prevents execution from proceeding: the two remaining 32-bit writes are later in the represented sequence.

The all-ones field is a 32-bit bit pattern. Its interpretation as negative one, an unsigned maximum, an invalid index, or another sentinel is not established.

This portion contains no direct writes to sixth-result offsets `+0x00` through `+0x07` or `+0x10` through `+0x13`. That absence is not a preservation guarantee: dependency effects or overlapping earlier destinations may affect those bytes.

### Final calls and return

After the fourth-input and all-ones writes, the function calls:

1. F-226d97c76fa5 with the derived address and the third input.
2. F-75e6a5ba5e57 with the same derived address.

Neither call’s return value is used. If both calls return normally, the function returns the derived address.

There are no further direct destination writes between these calls or after them. If either dependency changes an initialized field, this function does not restore that field.

The direct assignments establish values at their individual write points, not unconditional final-memory values. Dependencies may modify storage, change globals, or invalidate allocations. Their behavior must permit subsequent operations for normal completion to remain possible.

No visible branch, retry, conditional cleanup, or explicit error return occurs in the supplied body.

### Conditional contiguous layout

If each of the first six dependencies returns exactly its first argument, and the address arithmetic remains valid, their results have these offsets from the first input:

| Result | Offset |
|---|---:|
| First | `+0x08` |
| Second | `+0x14` |
| Third | `+0x24` |
| Fourth | `+0x30` |
| Fifth | `+0x40` |
| Sixth | `+0x54` |

Under that condition, subtracting `0x54` from the sixth result yields the original first-input address.

The direct writes then have the following layout:

| Offset from first input | Width | Assigned value |
|---|---|---|
| `+0x00` | 32 bits | Initial global snapshot |
| `+0x04` | 32 bits | Second input |
| `+0x3c` | 32 bits | Zero |
| `+0x5c` | 32 bits | Zero |
| `+0x60` through `+0x63` | Four ordered 8-bit writes | Zero |
| `+0x68` | 32 bits | Fourth input |
| `+0x6c` | 32 bits | `0xffffffff` |

These direct accesses require writable locations through offset `+0x6f`. A contiguous allocation covering them would span at least `0x70` bytes from the first input. This is not an established object size or a sufficient allocation requirement for the dependencies, whose accesses are unknown.

## Side effects

The direct side-effect inventory is:

- One 32-bit global read.
- Two 32-bit writes relative to the first input.
- One 32-bit zero write relative to the fourth dependency result.
- Three 32-bit writes relative to the sixth dependency result, at `+0x08`, `+0x14`, and `+0x18`.
- Four ordered 8-bit zero writes relative to the sixth dependency result, at `+0x0c` through `+0x0f`.
- Eight dependency invocations in the specified order.

In total, the body explicitly performs six 32-bit writes and four 8-bit writes. These counts describe operations, not necessarily distinct destinations: returned addresses and the initial destination may overlap.

No direct allocation, release, reference-count operation, synchronization operation, or I/O operation is visible. Any of those effects may occur inside dependencies.

No ownership transfer is established. Passing or returning an address does not prove that this function or a dependency retains, acquires, or releases the corresponding storage.

The function does not visibly save and restore global state around calls. In particular, the initial global snapshot does not imply that dependencies preserve the global location.

If an access fails or a dependency does not return normally, there is no visible rollback. Earlier writes and dependency effects may already have occurred. No atomicity or transactional guarantee is established.

## Dependencies

| Opaque ID | Established role |
|---|---|
| F-ad433e30ea59 | Receives first-input offset `+0x08` and zero; starts the returned-address chain |
| F-fa70d5b4a668 | Receives the first result plus `0x0c` and two zeros |
| F-7caeaed12e0c | Receives the second result plus `0x10` and zero |
| F-95094c640cf9 | Receives the third result plus `0x0c` and zero; returns the base for the intervening zero write |
| F-758c15808c75 | Receives the fourth result plus `0x10`, after that zero write |
| F-e511432e9f02 | Receives the fifth result plus `0x14`; returns the base for the remaining writes and return-address calculation |
| F-226d97c76fa5 | Receives the derived address and third input after all direct writes |
| F-75e6a5ba5e57 | Receives the same derived address immediately before normal return |

Dependency internals, storage requirements, ownership effects, global effects, and failure conventions are documented_only unknowns. No assumption that a dependency preserves earlier contents, preserves globals, or keeps storage alive is part of this contract.

## Acceptance examples

These are conditional acceptance criteria derived from static evidence. They are not executed tests.

### Identity-return chain

Given valid storage and normally returning dependencies that each return their first argument for the first six calls:

- The sixth result is the first input plus `0x54`.
- The returned address equals the first input.
- Direct writes occur at the offsets in the conditional contiguous layout.
- Both final dependencies receive the first-input address.
- F-226d97c76fa5 also receives the third input unchanged.

Final field contents additionally depend on dependency effects.

### Redirected chain

If F-ad433e30ea59 returns an address different from its argument, F-fa70d5b4a668 receives that returned address plus `0x0c`.

Likewise, a redirected fourth result determines both the intervening zero-write destination at `+0x0c` and the fifth dependency argument at `+0x10`. Continuing from an assumed original-object layout would not satisfy the established behavior.

### Sixth-result write trace

For a valid sixth result and normal progression, the direct trace relative to that result contains exactly:

- A 32-bit zero write at `+0x08`.
- Four successive 8-bit zero writes at `+0x0c`, `+0x0d`, `+0x0e`, and `+0x0f`.
- A 32-bit fourth-input write at `+0x14`.
- A 32-bit all-ones write at `+0x18`.

The return-address calculation occurs after the four byte writes and before the fourth-input write. An additional fourth 32-bit write relative to the sixth result is not supported by the body.

### Distinct input patterns

If the second and fourth inputs contain different 32-bit patterns, the second input is written at first-input offset `+0x04`, while the fourth input is written at sixth-result offset `+0x14`.

The third input is forwarded to F-226d97c76fa5. The body establishes no direct destination write of that input.

### Aliasing the global snapshot

Suppose the second-word destination aliases the global source location, and all accesses are valid. The function first snapshots the global value, then writes the second input to that location, then writes the saved snapshot at first-input offset `+0x00`.

The first-word value must therefore come from the earlier read, not from the newly written global contents.

### Zero result or abnormal completion

A zero-valued dependency result does not trigger a visible error branch. The next operation derives its address from zero using the specified displacement. Whether that leads to valid execution, a fault, or another outcome remains environment- and dependency-dependent.

If a dependency terminates execution nonlocally, later writes and calls are not guaranteed. Earlier effects are not visibly undone.

### Final dependency mutation

If F-226d97c76fa5 changes a previously initialized field and returns normally, no direct write restores that field before F-75e6a5ba5e57 runs. If both calls return normally, the function returns the previously derived address, not either dependency’s return value.

## Uncertainties

The following remain documented_only:

- The global value’s meaning and relationship to the destination.
- The original object type and field semantics.
- Whether dependency results normally equal their first arguments.
- Complete allocation sizes and alignment requirements.
- Byte order for multi-byte stores.
- Exact arithmetic behavior outside valid address ranges.
- Pointer provenance and original public interface details.
- Dependency errors, exceptions, traps, and nonlocal exits.
- Allocation, release, retention, and lifetime effects inside dependencies.
- Thread safety, reentrancy, atomicity, and hardware memory ordering.

Constructor-like intent and possible sentinel meanings are inferences, not established behavior. The specified ordering is the order of operations in the supplied representation, not an independent guarantee about concurrent observation.

## Evidence

This contract is based solely on the supplied static function body and opaque dependency mapping. The body establishes the access widths, address displacements, ordered writes, call sequence, argument forwarding, and returned-address calculation.

The corrected side-effect count follows directly from the three explicit 32-bit writes at sixth-result offsets `+0x08`, `+0x14`, and `+0x18`; the four intervening byte writes are counted separately.

The contiguous layout is conditional arithmetic, not an asserted dependency contract. No dependency inspection, implementation access beyond the supplied material, runtime execution, or tests have been performed.
