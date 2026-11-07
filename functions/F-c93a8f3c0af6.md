<!-- Trusted provenance: F-c93a8f3c0af6; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-c93a8f3c0af6

## Purpose

F-c93a8f3c0af6 modifies a collection represented by three address-bearing fields and eight-byte records. Its inputs identify the descriptor, an insertion position, and a source record.

Two paths are established by the supplied representation:

- When the initial capacity count is no greater than the initial element count, the function obtains an address from a dependency, invokes dependencies with range-like and destination arguments, calls another dependency with a retained storage address and count, and then publishes replacement descriptor fields.
- Otherwise, it advances the descriptor’s end by eight bytes, invokes a dependency at the former end, shifts selected existing records one slot toward higher addresses, and writes the supplied record at the insertion position.

**Inferred intent:** insertion into a growable sequence, with storage acquisition, record transfer, and eventual release of previous storage on the replacement path.

Those roles are not established dependency semantics. In particular, the call involving retained storage does not itself prove that storage is released.

The spare-capacity path also does not independently establish ordinary insertion semantics: its direct copy loop never writes the newly exposed slot at the former end. That slot’s contents depend on the dependency call or on storage contents already present.

## Interface

### Inputs and representation

The function receives three addresses:

| Input | Established use |
|---|---|
| Descriptor address | Locates three consecutive four-byte fields containing storage addresses |
| Insertion position | Supplies a range endpoint, determines replacement destinations, and identifies the final direct-write destination on the spare-capacity path |
| Value-source address | Identifies two consecutive four-byte words, or is passed to a dependency |

The descriptor fields are:

| Byte offset | Descriptive name | Interpretation |
|---|---|---|
| 0 | Begin | Storage beginning |
| 4 | End | Logical end |
| 8 | Limit | Storage limit |

Addresses and integer fields are represented at 32-bit width. Each record occupies eight bytes: a first 32-bit word followed by a second 32-bit word. No numeric, pointer, ownership, or resource interpretation of either record word is established.

“Original position” and “original value source” below mean the addresses supplied as arguments. They do not imply snapshots of the memory at those addresses. “Initial end” means the End value retained before branch selection.

### Output

No explicit return value is established. Observable results consist of:

- Descriptor and record writes.
- Dependency calls and their arguments.
- Any additional effects of those dependencies.
- Failure or noncompletion caused by inaccessible memory, unsuitable arithmetic, invalid positions, or dependency behavior.

### Preconditions for ordinary bounded operation

The function does not enforce the following conditions:

- The descriptor is accessible for its required reads and writes.
- Descriptor addresses describe compatible storage with appropriate ordering and eight-byte spacing.
- All directly accessed words are accessible.
- Calculated addresses and storage sizes are representable and usable.
- Dependencies accept the supplied arguments and return normally when continued execution is expected.

For a nonempty, ordinarily ordered collection on the spare-capacity path, the insertion position must be an existing record address from Begin through Initial end minus eight, inclusive. The backward traversal starts at Initial end minus eight and proceeds toward lower addresses.

A position equal to Initial end is not in that ordinary terminating range. An empty collection with spare capacity has no existing-record position satisfying it.

No aliasing exclusion is checked. Aliasing among the descriptor, value source, existing records, and dependency-accessible storage can change subsequent reads or state.

## Behavior

### Initial inspection and path selection

The function retains End before making any dependency call.

It derives an initial element count by subtracting Begin from the retained End, interpreting the subtraction and right shift as signed operations in the supplied representation, shifting right by three, and retaining the result as an unsigned 32-bit count.

It separately derives a capacity count from Limit minus Begin, shifted right by three, and compares that result as unsigned against the initial element count.

The replacement path is selected when:

**initial capacity count ≤ initial element count**

The spare-capacity path is selected otherwise.

With ordered addresses, eight-byte spacing, and representable differences, these counts correspond to used records and total record slots. Outside those conditions, the signed arithmetic and subsequent unsigned interpretation need not match mathematical nonnegative counts.

### Replacement path: capacity calculation

Two candidate capacities are calculated from the initial unsigned element count:

- Initial count plus 32.
- Initial count plus its unsigned half, rounded down, plus its unsigned eighth, rounded down.

The greater unsigned candidate is selected. Equality selects the same numerical capacity regardless of which candidate is retained.

The additions operate at 32-bit unsigned width and may wrap. The selected capacity is shifted left by three to form a 32-bit byte-size argument; high bits may be discarded. No overflow check or maximum-capacity check is visible.

Without overflow, the policy selects at least 32 additional slots or approximately 62.5% additional slots, whichever gives the larger capacity.

### Replacement path: dependency calls

The following calls occur in order:

1. **F-871e64ddc6de** receives the selected capacity shifted left by three and a second argument of zero. Its returned address is retained as the replacement base.
2. **F-b788411aeb24** receives the descriptor’s then-current Begin, the original position, the replacement base, and the descriptor address.
3. **F-c3d28e48088c** receives the descriptor address, a calculated replacement destination, and the original value-source address.
4. **F-b788411aeb24** receives the original position, the descriptor’s then-current End, a second calculated replacement destination, and the descriptor address.

For the third call, the destination is the replacement base plus the difference between the original position and the descriptor’s then-current Begin, with the low three bits of that difference cleared by the 32-bit mask `0xfffffff8`.

For the fourth call, the destination is calculated again using the then-current Begin, with eight additional bytes added.

For a valid aligned relative position, clearing the low three bits does not change the offset. For an unaligned relative position, it changes the offset; it is not a bounds or validity check.

Descriptor reads are not all taken from an immutable entry snapshot:

- The Begin used by the first range-like call is read after F-871e64ddc6de returns.
- The destination for the next call uses Begin read after the first range-like call returns.
- The final range-like call uses descriptor fields read after F-c3d28e48088c returns.

A dependency may therefore change arguments supplied to later dependencies. Exact ordering among separate field reads used to prepare one call is not established beyond the supplied expression dependencies.

### Replacement path: retained endpoints and traversal

After the population-related calls return, the function captures the descriptor’s current Begin and End.

A local traversal then advances from that retained Begin by eight bytes until it equals the retained End. Its body contains no visible memory access or dependency call.

When the endpoints are compatible, the traversal has no visible per-record action. It does not establish destruction, cleanup, or ownership transfer.

If the endpoints cannot meet through the represented eight-byte progression, ordinary bounded completion is not established. Whether this side-effect-free traversal survives compilation as a runtime loop is also unresolved.

### Replacement path: final dependency and publication

After the traversal, **F-ff1bdc376dcd** receives:

- The retained Begin.
- The signed difference between the descriptor’s current Limit and the retained Begin, shifted right by three.
- Zero.

The Limit used here is read at this stage, rather than retained from entry. The call’s effect on storage ownership or lifetime is unknown.

If that dependency returns, the function writes the descriptor in this order:

1. Begin becomes the replacement base.
2. End becomes the replacement base plus eight times the count derived from the retained End minus retained Begin, plus eight.
3. Limit becomes the replacement base plus eight times the selected capacity.

The count used to publish End is derived from endpoints captured after the population-related calls. It is not necessarily the initial unsigned count used for capacity selection.

These writes are separate field updates, not an atomic descriptor replacement. Descriptor changes made by the final dependency are overwritten in these three fields; other dependency effects remain unconstrained.

### Spare-capacity path: early End update

The function writes End as Initial end plus eight before invoking any dependency on this path.

It then calls **F-c3d28e48088c** with two visible arguments:

- The descriptor address.
- Initial end.

The replacement path supplies three visible arguments to the same dependency. Its exact signature and the interpretation of the differing argument lists remain unresolved.

The dependency can observe the advanced End. There is no visible rollback if it fails, does not return normally, or leaves memory unsuitable for subsequent operations.

### Spare-capacity path: backward record movement

After the dependency returns, the first candidate destination is Initial end minus eight. If that address equals the original position, no records are copied.

Otherwise, the function copies the record immediately preceding that destination into the destination, then continues with destinations decreasing by eight until the destination reaches the original position.

Every iteration has this exact visible word-access order:

1. Read the source record’s first 32-bit word and retain it.
2. Read the source record’s second 32-bit word.
3. Write that second word to the destination’s second word.
4. Write the retained first word to the destination’s first word.

Thus, the first source word is read before the second source word. The destination’s second word is written before its first word. The record transfer is not indivisible.

For an ordinary valid position, the loop visits destination slots from Initial end minus eight down through Position plus eight. Each selected source record moves one slot toward higher addresses.

The slot at Initial end is never a destination of this loop. The original last record is therefore not explicitly copied into that slot by the visible loop.

Traversal uses the retained Initial end and original position, not descriptor bounds reread after the dependency.

### Spare-capacity path: supplied-record write

After the backward movement terminates, the function performs these accesses:

1. Read the value source’s second word and retain it.
2. Read the value source’s first word.
3. Write that first word at the original position.
4. Write the retained second word four bytes after the original position.

Unlike the backward-copy loop, this final operation reads the second source word before the first.

The supplied record is read after all backward movement, not captured at entry. If movement or dependency activity changes the value-source memory, the final write uses those changed contents.

## Side effects

### Descriptor and storage

The spare-capacity path explicitly changes only End, before its dependency call. It does not subsequently restore Begin, End, or Limit if the dependency changes them.

The replacement path explicitly publishes all three descriptor fields after F-ff1bdc376dcd returns.

Only the spare-capacity path contains direct record reads and writes. Replacement-storage population is delegated to dependencies.

Dependencies receiving the descriptor may mutate it. Dependencies not receiving it directly are not assumed to preserve it or other global state.

### Ownership and lifetime

The replacement path obtains an address from one dependency and later passes retained storage information to another.

**Inferred intent:** acquire replacement storage and relinquish previous storage.

**Not established:** whether either dependency allocates or deallocates memory, who owns returned storage, whether record transfer copies or moves resources, whether old records are destroyed, or whether ownership changes at all.

No explicit storage-disposition dependency call appears on the spare-capacity path, but this does not constrain the internal effects of F-c3d28e48088c.

### Errors and incomplete execution

There is no visible status result, null check, allocation-failure branch, position check, arithmetic-overflow check, exception recovery, or rollback.

An unusable replacement address is not rejected before being passed onward. If a dependency does not return normally, later operations are not reached; no cleanup behavior is established.

Invalid positions, inconsistent descriptors, and inaccessible addresses may cause invalid accesses, unsuitable dependency arguments, or noncompletion. No particular failure mechanism is guaranteed.

## Dependencies

| Opaque ID | Established interaction | Unresolved contract |
|---|---|---|
| F-871e64ddc6de | Receives a 32-bit byte-size value and zero; returns the replacement address | Allocation, alignment, failure signaling, ownership, descriptor and global effects |
| F-b788411aeb24 | Called twice with range-like addresses, a destination, and the descriptor | Endpoint conventions, copy versus move, overlap, descriptor mutation, resource effects |
| F-c3d28e48088c | Receives three visible arguments on the replacement path and two on the spare-capacity path | Exact signature, record initialization or assignment, handling of the former last record, state and ownership effects |
| F-ff1bdc376dcd | Receives retained Begin, a signed eight-byte-unit count, and zero before descriptor publication | Whether storage is released, count interpretation, failure behavior, ownership and global effects |

No dependency is assumed to preserve globals, preserve descriptor fields, or acquire or release resources merely because its call resembles a familiar operation.

## Acceptance examples

These examples are conditional static consequences, not observed executions.

### Full collection containing four records

Assume ordered, aligned storage describing four used slots and four total slots, with no arithmetic overflow.

- The replacement path is selected.
- Capacity candidates are 36 and 6.
- Selected capacity is 36.
- F-871e64ddc6de receives 288 and zero.

If dependencies return normally and preserve the relevant descriptor fields until publication, final End is replacement base plus 40 and final Limit is replacement base plus 288.

Record preservation and disposition of the old storage remain dependency-dependent.

### Full collection containing 64 records

Under the same arithmetic and descriptor assumptions:

- Capacity candidates are 96 and 104.
- Selected capacity is 104.
- The byte-size argument is 832.
- If the retained post-population endpoints still describe 64 records, published End describes 65 records.

### Empty collection with no capacity

Assume Begin, End, and Limit are equal and all subsequent dependency interactions are valid.

- Both initial counts are zero.
- The replacement path is selected.
- Selected capacity is 32; the byte-size argument is 256.
- If the retained post-population endpoints remain equal, published End is replacement base plus eight.

Whether the dependencies accept the empty ranges and equal endpoint addresses is unresolved.

### Spare capacity, position at the former last record

Assume three records and position at Begin plus 16.

- End advances from Begin plus 24 to Begin plus 32.
- F-c3d28e48088c receives the descriptor and former End.
- No backward-copy iteration occurs.
- The supplied record overwrites the former last record.

The fourth slot’s contents are not established independently of the dependency and preexisting storage.

### Spare capacity, position at the first record

Assume three records, position at Begin, and a dependency that leaves the existing records unchanged.

The loop first copies the second record into the third slot, then the first record into the second slot. Each copy reads the first source word, reads the second, writes the second destination word, and writes the retained first destination word.

The supplied record is then written into the first slot. The fourth slot remains outside the direct copy loop.

### Value source aliases a shifted record

Use the preceding case, with the value source at the original second slot and no conflicting descriptor aliasing.

Before the final source reads, movement has replaced that slot with the original first record. The final insertion therefore uses the original first record’s words, not the second record’s entry contents.

### Spare capacity, position equals Initial end

The first candidate destination is Initial end minus eight, and subsequent candidates move backward. Initial end is outside the ordinary valid terminating range.

This case must not be accepted as an established append operation. Any behavior relying on address wraparound lies outside the bounded-storage assumptions.

## Uncertainties

The following remain **documented_only**:

- Exact dependency signatures and all internal dependency effects.
- How the newly exposed spare-capacity slot obtains its contents.
- Whether any dependency copies, moves, constructs, destroys, allocates, or releases.
- Dependency failure signaling and nonlocal control flow.
- Caller guarantees concerning positions, empty collections, alignment, and aliasing.
- Behavior outside representable, compatible address arithmetic.
- Runtime treatment of the side-effect-free traversal.
- Portable language-level semantics for signed subtraction overflow, negative right shifts, and address conversions in the supplied representation.

The explicit unsigned growth arithmetic has 32-bit wrapping behavior. That does not establish portable wrapping semantics for every signed address calculation.

## Evidence

This contract is based solely on static inspection of the supplied function representation. No runtime tests have been performed, and no dependency implementation was available.

Established evidence includes the three-field descriptor layout, eight-byte stride, unsigned path comparison, growth calculations, dependency sequence, descriptor rereads, early End update, distinct word-access orders for movement and final insertion, side-effect-free traversal, and ordered final descriptor writes.

Growable-sequence insertion, allocation, transfer, and release are inferred roles only where identified. Neither storage release nor preservation of the former last record is established by the visible function alone.
