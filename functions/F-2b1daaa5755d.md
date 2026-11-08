<!-- Trusted provenance: F-2b1daaa5755d; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-2b1daaa5755d

## Purpose

F-2b1daaa5755d performs a lookup and, when the lookup reports absence, attempts to construct and insert an entry. Both paths finish by calling a result-reporting dependency with a computed placement value and a 32-bit flag distinguishing the existing-entry path from the insertion path.

The lookup-or-insert interpretation is inferred from the control flow. The established behavior is narrower: a null dependency result selects allocation, possible growth, and insertion; a non-null result bypasses those operations. The function does not itself establish allocation success, insertion success, uniqueness, or the meaning of the reported result.

No runtime tests have been performed.

## Interface

### Inputs

The function accepts three machine-level arguments:

| Argument | Width or representation | Established use |
|---|---|---|
| Result destination | 32-bit opaque value | Saved at entry and passed to F-8693ded0dc82 on either completion path. This function does not directly dereference it. |
| Container | 32-bit address | Used for field reads, address formation, and calls to lookup, placement, growth, and insertion dependencies. |
| Key descriptor | 32-bit address | Supplies two 32-bit words and is passed as an address to the lookup dependency. On the insertion path, its words are read again after allocation. |

The key descriptor must provide readable words at byte offsets 0 and 4 whenever those reads occur. The function does not take an independent, durable snapshot of both words at entry.

The directly observed container fields are:

| Byte offset | Access width | Use |
|---|---:|---|
| 4 | 32 bits | Relative displacement used to form the first argument to F-58cc241cfb6b. |
| 8 | 32 bits | Argument to F-0ec3a2a1b51e on the growth path. |
| 12 | 32 bits, unsigned comparison | Compared with the incremented value derived from offset 32. |
| 16 | Address formation | Base address passed to placement and insertion dependencies. No direct field read at this offset is shown. |
| 32 | 32 bits, unsigned arithmetic | Incremented and used to calculate whether growth is requested and the proposed growth value. |

The conventional interpretation of offsets 12 and 32 as a capacity-like bound and entry count is inferred, not independently established.

### Output

There is no explicit return value.

On each normally completing path, the function calls F-8693ded0dc82 with:

1. The saved result-destination value.
2. The address of a temporary 32-bit placement value.
3. The address of a temporary 32-bit flag.

The flag is 0 for a non-null lookup result and 1 for the insertion path. It is not a checked success status.

A temporary candidate pointer is stored immediately after the placement word in the supplied stack layout. It contains either the lookup result or the insertion candidate. This supports an inference that the reporting dependency may consume a two-word descriptor beginning at the placement word. The dependency’s actual read extent is unknown; only the address of the placement word is explicitly passed.

### Preconditions

For normal completion:

- Container fields read by this function must be accessible at the time of each read.
- The key descriptor must remain accessible through any later rereads.
- A non-null allocation result must identify writable storage sufficient for the writes described below.
- Dependencies must accept the arguments passed to them, including a null insertion candidate if allocation fails.
- Dependencies receiving temporary addresses must consume them within their valid lifetime unless their own contract provides another mechanism.

No input validation or local error recovery is visible. Overlap, concurrent mutation, callback-driven mutation, and reentrancy guarantees are not established.

## Behavior

### Initial computation and lookup

The function first saves the incoming arguments in temporary storage.

It obtains a 32-bit displacement from container offset 4 and forms an address by adding that displacement to the container address. It then calls F-58cc241cfb6b with:

- The resulting address.
- The container address.
- The first 32-bit key word.
- The second 32-bit key word.

The relative ordering of the field read and the two key-word reads within this call’s argument preparation is not established by the supplied representation.

The 32-bit result of F-58cc241cfb6b is retained for possible reuse after growth. It is passed to F-e9400ed1bb91 together with the address at container offset 16.

The resulting 32-bit placement value is then passed to F-1cf336eed072, along with the container address and the original key-descriptor address.

The nullness of F-1cf336eed072’s returned pointer determines the branch. No other property of that pointer is inspected directly.

### Existing-entry path

If the lookup result is non-null:

- No allocation is requested by this function.
- The growth-related container fields are not read by this function.
- No growth or insertion dependency is called.
- The temporary candidate pointer remains the lookup result.
- The report flag is set to 0.
- The report placement word is set to the value computed before lookup.
- F-8693ded0dc82 is called.

The function then returns, provided the reporting dependency returns normally.

This branch does not recompute placement after lookup. Any dependency-induced mutation does not cause this function to refresh that saved placement value.

### Missing-entry path: allocation and initialization

If the lookup result is null, the function calls F-871e64ddc6de with a requested size of 16 bytes and a second argument of 0.

For a non-null returned address, direct initialization occurs in this order:

1. Write a 32-bit zero at allocation offset 0.
2. Write a 32-bit zero at allocation offset 4.
3. Write a 32-bit zero at allocation offset 8.
4. Read the second 32-bit key word.
5. Read the first 32-bit key word and write it at allocation offset 0.
6. Write the saved second key word at allocation offset 4.

There is no direct initialization of allocation offsets 12–15.

The final directly established layout, before subsequent dependency calls, is therefore:

| Allocation byte range | Directly established contents |
|---|---|
| 0–3 | First key word reread after allocation |
| 4–7 | Second key word reread after allocation |
| 8–11 | Zero |
| 12–15 | Not initialized by this function |

The key values copied into the allocation need not equal the values supplied to the initial computation if intervening dependencies or external activity change the descriptor. No consistency check is performed.

For a null allocation result, all these initialization writes and key rereads are skipped. The function nevertheless continues to the growth decision and insertion call.

### Growth decision and arithmetic

After allocation and any initialization, the function reads the 32-bit unsigned value at container offset 32. It computes that value plus one using 32-bit unsigned arithmetic.

It then reads the 32-bit value at container offset 12. Growth is selected when that value is less than or equal to the incremented value.

Thus, equality selects growth. This is not a strictly-less-than test.

All additions in the unsigned growth calculation wrap modulo \(2^{32}\). No overflow guard or saturation behavior is visible.

When growth is selected, the proposed value is the larger, under unsigned comparison, of:

- The original offset-32 value plus its logical right shift by one, with the addition reduced modulo \(2^{32}\).
- The incremented offset-32 value.

This resembles a one-and-a-half-times growth rule, but wrapping can make the arithmetic depart from that interpretation.

### Growth dependency sequence

On the growth branch, the following calls occur in order:

1. F-0ec3a2a1b51e receives the current 32-bit field at container offset 8 and produces a 64-bit result.
2. F-9e5e664c8ae7 receives the proposed growth value and produces another 64-bit result.
3. F-596e18a33aec receives four 32-bit arguments: the low and high halves of the second result, followed by the low and high halves of the first result.
4. F-936416edf8a5 is called without explicit arguments.
5. F-1cd4c8db3fc1 is called without explicit arguments.
6. F-72dd99163ef1 receives the container address and the preceding call’s result plus one.

The result of F-1cd4c8db3fc1 is represented as a signed 32-bit integer. The supplied material does not establish a portable arithmetic contract for adding one when that result is the largest signed 32-bit integer. That boundary is documented_only.

The first 64-bit result is retained across the second dependency call. Both 64-bit values are explicitly split into 32-bit halves for F-596e18a33aec; no floating-point interpretation is established.

After F-72dd99163ef1 returns, F-e9400ed1bb91 is called again with the container-offset-16 address and the originally retained result of F-58cc241cfb6b. The newly returned placement value replaces the earlier one.

The function does not repeat lookup after growth and does not recalculate the original key-derived value.

The saved growth decision controls this recomputation. It is not reevaluated after the growth dependencies run.

### Insertion and reporting

After the growth branch, or immediately when growth was not selected, the function derives an insertion candidate:

- A non-null allocation address is advanced by 8 bytes.
- A null allocation address remains null.

F-49c170fffc12 then receives:

1. The address at container offset 16.
2. The insertion candidate.
3. The current placement value.

Thus, a successful allocation supplies a pointer into the allocation rather than its base. The two copied key words lie 8 and 4 bytes before that candidate address. The zeroed word at allocation offset 8 begins at the candidate address.

After the insertion call returns:

- The report flag is set to 1.
- The report placement word is set to the current placement value.
- The temporary candidate pointer is set to the insertion candidate.
- F-8693ded0dc82 is called.

No insertion result is examined. In particular, the flag remains 1 when allocation returned null, provided all intervening calls return normally.

## Side effects

### Direct memory effects

The only direct non-temporary writes shown are initialization of a successful 16-byte allocation.

This function does not directly write the observed container fields, increment the offset-32 field, initialize the final four bytes of the allocation, or write through the result-destination argument.

Any such effects must come from dependencies.

### Dependency-mediated effects

The lookup, growth, insertion, allocation, and reporting dependencies may have observable effects. Their names and argument shapes do not prove that they are pure or that they preserve global state, the container, the key descriptor, or other reachable memory.

Reads occur at the stages described above. In particular:

- Initial key reads precede lookup.
- Allocation precedes the insertion-path key rereads.
- Growth-related field reads occur after allocation.
- The offset-8 field is read only on the growth branch.
- Placement is recomputed after growth but not after insertion.

These distinctions matter when dependencies mutate shared state.

### Ownership and lifetime

The function requests storage but does not explicitly release it.

Passing an interior pointer to F-49c170fffc12 does not establish ownership transfer or identify who must eventually release the base allocation. Cleanup on insertion failure, dependency failure, or reporting failure is unknown.

The reporting dependency receives temporary addresses. Their retention, interpretation, and required lifetime are not established.

### Errors

There is no explicit error return, rollback, retry, or failure cleanup.

Allocation failure is observable as a null candidate passed onward, not as an early exit. Whether later dependencies tolerate that candidate is unknown.

Non-returning behavior, faults, and exceptions from dependencies are not handled locally in the supplied material.

## Dependencies

| Opaque ID | Established role at this call site |
|---|---|
| F-58cc241cfb6b | Produces a retained 32-bit value from a relative address, the container, and two key words. |
| F-e9400ed1bb91 | Produces a placement value before lookup and, conditionally, after growth. |
| F-1cf336eed072 | Returns the pointer whose nullness selects the branch. |
| F-871e64ddc6de | Receives a 16-byte request and zero; its result is treated as an allocation address. |
| F-0ec3a2a1b51e | Produces a 64-bit intermediate from container offset 8. |
| F-9e5e664c8ae7 | Produces a 64-bit intermediate from the proposed growth value. |
| F-596e18a33aec | Consumes the two split 64-bit intermediates in the specified order. |
| F-936416edf8a5 | No-argument operation within the growth sequence. |
| F-1cd4c8db3fc1 | Supplies the integer used, after increment, as the growth helper’s second argument. |
| F-72dd99163ef1 | Receives the container and that incremented integer. |
| F-49c170fffc12 | Receives the container substructure address, insertion candidate, and placement value. |
| F-8693ded0dc82 | Receives the saved destination and addresses of temporary result information. |

These are call-site roles, not independent contracts for the dependencies.

## Acceptance examples

The following are conditional acceptance cases, not executed tests. Each assumes valid memory and normally returning dependencies unless stated otherwise.

### Existing entry

Given a non-null lookup result, the observable call sequence ends with reporting. There is no allocation, growth sequence, or insertion call. The reporting flag is 0, and the placement word is the original placement result.

### Missing entry without growth

Given an offset-32 value of 3 and an offset-12 value of 10, the incremented value is 4 and growth is not selected.

If allocation succeeds, its first two words receive the reread key words, its third word is zeroed, and its fourth word is not directly initialized. Insertion receives the allocation address plus 8 and the original placement value. Reporting receives flag 1.

### Equality triggers growth

Given an offset-32 value of 3 and an offset-12 value of 4, growth is selected because 4 is less than or equal to 4.

The proposed growth value is 4. The full growth dependency sequence occurs before placement is recomputed and insertion is attempted.

### Ordinary growth calculation

Given an offset-32 value of 10 and an offset-12 value of 11, growth is selected. The incremented value is 11, while the one-and-a-half calculation produces 15. F-9e5e664c8ae7 receives 15.

### Allocation failure

Given a null lookup result followed by a null allocation result, no allocation initialization occurs. Growth is still considered. If all subsequent dependencies return, insertion receives a null candidate and reporting receives flag 1.

This case must not be described as successful insertion solely because of that flag.

### Unsigned wraparound

Given an offset-32 value of \(2^{32}-1\), its incremented value is zero. A positive offset-12 value therefore does not select growth; an offset-12 value of zero does.

This follows the observed unsigned comparison rather than an unbounded mathematical count.

### Descriptor mutation

If a dependency changes the key descriptor before allocation initialization, the copied words come from the later reads. The retained initial computation is not refreshed to match those new words.

## Uncertainties

The following are documented_only:

- Semantic meanings of container fields and dependency results.
- Whether the structure is a hash-based container or another indexed representation.
- Whether the reporter reads the candidate pointer adjacent to the placement word.
- Ownership of successful allocations and cleanup responsibility.
- Null-candidate support in insertion.
- Global-state effects of the no-argument growth operations.
- Dependency mutation, reentrancy, and synchronization guarantees.
- Endianness and complete structure layouts.
- Address-overflow behavior and validity requirements for relative address formation.
- The signed increment boundary following F-1cd4c8db3fc1.
- Runtime reachability and behavior outside the supplied static representation.

## Evidence

This contract is based solely on the supplied static function representation and opaque dependency mapping.

Established evidence includes call order, branch conditions, direct memory accesses, 32-bit unsigned growth arithmetic, 64-bit result splitting, allocation initialization, interior-pointer formation, and reporting flags.

Lookup-or-insert terminology, capacity/count terminology, ownership transfer, and a two-word reporting descriptor remain interpretations where explicitly identified. Embedded comments are not treated as independent evidence.

No runtime tests, execution traces, or independent dependency contracts were supplied or used.
