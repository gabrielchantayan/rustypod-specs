<!-- Trusted provenance: F-e7dcc511f75e; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-e7dcc511f75e

## Purpose

F-e7dcc511f75e attempts to place a derived 32-bit candidate into the first available entry of an associated table. Ordinary completion returns the selected zero-based index, or −1 when the candidate is zero, an existing duplicate is found, or the available-slot search fails.

Rejection is not side-effect-free. Two construction-related writes precede every explicit rejection. Dependencies may also have effects that this function neither reverses nor reports.

**Inferred intent:** the operation resembles registration of a newly constructed item. The evidence does not establish object semantics, allocation ownership, or a general uniqueness guarantee. In particular, an intervening dependency can change the table after duplicate detection.

## Interface

### Inputs and return value

The function accepts two opaque 32-bit arguments:

- **First argument:** passed unchanged to F-b9acea4b06ad as its second argument.
- **Second argument:** written as a four-byte value to a destination derived from the result of F-b9acea4b06ad.

Neither argument has an established domain-specific type, ownership role, or validation rule.

The return value is a signed 32-bit integer:

| Result | Established meaning |
|---|---|
| −1 | One of the explicit rejection conditions occurred, or no available slot was found. |
| Nonnegative index | A candidate-placement write completed at that index. The index is strictly below the length captured for slot selection. |

A successful return is an index, not an address or an established owning handle. The function provides no distinct return codes for the different rejection causes.

### Construction-related storage

F-8157a180d901 is called with the numeric request 16. Its result determines the destination of a four-byte initialization write and an interior address four bytes farther on. That interior address and the first input are supplied to F-b9acea4b06ad.

The latter dependency’s result determines two locations:

- The candidate is four bytes before that result.
- The second-input destination is eight bytes after that result.

Consequently, when these address relationships are valid, the second input is stored twelve bytes after the candidate.

The evidence does not require F-b9acea4b06ad to return the interior address it received. Nor does the request value 16 prove that either dependency supplies sixteen accessible bytes. These address relationships describe required accesses, not established storage guarantees.

### Associated record layout

For a nonzero candidate, F-ecd921e533b1 supplies an associated-record address. The function uses three four-byte fields:

| Byte offset | Representation and role |
|---|---|
| 96 | Address-like table-base value. The address of this field is supplied to F-e901f09a5170 when that dependency is invoked. |
| 100 | Signed 32-bit search length. |
| 104 | Signed 32-bit field compared with the search length and incremented before successful placement. |

Each table entry is four bytes wide. Entry comparisons concern equality with the candidate or zero; no signed ordering of entry values is required.

“Search length” is an operational description, not proof of allocated capacity. Likewise, offset 104 is not established to be an occupancy count, capacity, or number of live entries.

### Conditions for ordinary completion

Every executed access must address valid, suitably accessible storage. Relevant arithmetic must also be valid, including candidate derivation, address offsets, table indexing, and the signed increment at offset 104.

There are no visible checks establishing successful allocation, a valid construction result, a valid associated record, or table storage sufficient for the search range. Violating these requirements does not have a specified clean-error outcome. In particular, −1 is not a general memory-failure indication.

## Behavior

### Rejection and commitment boundaries

The function has three explicit rejection conditions:

1. The derived candidate is zero.
2. Duplicate detection encounters an entry equal to the candidate.
3. Available-slot selection finds no zero entry within its captured range.

All three occur after the construction-related initialization and second-input writes. The second-input destination is not protected by the zero-candidate check.

A zero candidate prevents associated-record lookup and all subsequent table processing. For example, under ordinary nonwrapping address arithmetic, a construction result numerically equal to four derives a zero candidate. A construction result of zero instead derives −4; it does not satisfy the zero-candidate condition. Whether the preceding write is valid remains a separate requirement.

No visible rollback accompanies any rejection.

### Duplicate detection

For a nonzero candidate, duplicate detection inspects table entries in ascending index order beginning at zero. Any matching entry causes immediate rejection.

A zero entry has no special stopping effect during this search. An empty entry before a duplicate therefore does not permit insertion.

The duplicate search does not use a single captured length. Its continuation decisions consult the record’s signed length field repeatedly. Under stable state, a positive length covers exactly the indices below that length; a zero or negative length produces no entry reads.

The table-base field supplies the base used for entry accesses. The static representation does not establish one immutable table-base snapshot for the entire operation.

These access relationships do not define a supported concurrent-mutation protocol. They describe the supplied static behavior, not a guaranteed number of physical memory transactions.

### Conditional intervention

If duplicate detection finishes without a match, equality between the signed fields at offsets 104 and 100 determines whether F-e901f09a5170 is called.

The dependency receives the address of the table-base field at offset 96, not the table-base value itself. Its return value does not affect the caller’s decision.

The call is not restricted to positive lengths. Equal zero values or equal negative values also satisfy its condition.

No preservation contract is available for this dependency. It may affect the record, table, storage lifetime, globals, or other state. The call must not be interpreted as proven growth, successful allocation, or assurance of an available slot.

Duplicate detection is not repeated afterward. If the dependency introduces the candidate into the table, that new duplicate does not independently prevent a subsequent insertion into another zero entry.

### Available-slot selection

The available-slot search uses the signed length observed after the optional dependency has returned, or after its call has been skipped. That one captured value bounds this entire search.

Selection chooses the first zero entry in ascending index order. Later entries are not inspected once such an entry is found.

A nonpositive captured length yields −1 without table-entry inspection. A positive captured length with no zero entry also yields −1. Neither case performs the direct increment at offset 104 or the candidate-placement write.

This is an important distinction between the searches: duplicate detection uses repeated length observations, whereas slot selection uses one later length snapshot.

### Successful placement

A selected slot has a nonnegative index strictly below the captured length. Index zero is eligible; it is not a failure sentinel.

Successful placement has an observable write order:

- The current signed value at offset 104 is read, incremented by one, and written back.
- The table-base value used for placement is obtained after that counter-field write.
- The candidate is written to the selected entry.
- The selected index is returned.

There is no second emptiness check immediately before placement. The selected entry was observed as zero during the search, not necessarily at the instant of the final write.

There is no dedicated assignment to the length field at offset 100. This does not guarantee that the length remains unchanged when table storage or other destinations overlap the record.

The updates are not transactional. If execution fails after the increment but before placement completes, no compensating action is established.

## Side effects

### Direct memory effects

For executions reaching an explicit normal result, the direct construction-related writes are:

- A four-byte global initializer value copied to the address returned by F-8157a180d901.
- The second input copied to the destination derived from F-b9acea4b06ad’s result.

The initializer is sampled after F-8157a180d901 returns and before F-b9acea4b06ad is called. Its value must not be assumed to equal the value that existed before the first dependency call.

A successful placement additionally writes the incremented field at record offset 104, then writes the candidate into the selected entry.

Searches read record fields and table entries only as far as their respective stopping conditions require. Exact hardware-level access counts, volatile semantics, and memory-mapped-device behavior are not established.

### Aliasing

No general non-overlap guarantee is supplied for construction destinations, the initializer, the associated record, or the table.

The three record fields occupy distinct four-byte locations under ordinary valid address layout. Nevertheless, a table entry may overlap one of those fields, or a construction destination may overlap storage used later.

Consequences include:

- A construction-related write can change state later consumed by lookup or searching.
- A candidate-placement write can overwrite a record field if the selected table entry aliases it.
- When the selected entry aliases offset 104, placement can overwrite the increment just performed.

Thus, “offset 104 is incremented before placement” does not imply that its final value is always its previous value plus one.

### Ownership, lifetime, and synchronization

No explicit release operation appears in this function. That does not establish either a leak or retained ownership: dependency retention, transfer, and release behavior are unknown.

Neither return outcome grants established ownership or lifetime rights over construction storage, the candidate, the record, or the table.

No locking, atomic insertion, rollback, or supported concurrent-access contract is established. Callers must not treat the operation as an atomic uniqueness check followed by insertion.

## Dependencies

| Opaque function ID | Established interaction | Unresolved behavior |
|---|---|---|
| F-8157a180d901 | Receives 16. Its result supplies the initialization destination and the basis of the interior address passed onward. | Request units, accessible extent, failure signaling, allocation semantics, ownership, and global effects. |
| F-b9acea4b06ad | Receives the interior address and first input. Its result determines the candidate and second-input destination. | Result validity, relation to supplied storage, construction semantics, retention, release, and other effects. |
| F-ecd921e533b1 | Receives a nonzero candidate. Its result supplies the associated-record address. | Lookup meaning, record validity and lifetime, ownership, and state changes. |
| F-e901f09a5170 | Receives the address of the table-base field when offsets 104 and 100 compare equal after duplicate detection. Its return value is unused. | Effects on record fields, table contents, storage, ownership, globals, and success or failure signaling. |

No dependency is assumed to preserve globals, retain storage ownership, or keep previously obtained storage alive. Continued execution requires the accesses actually performed afterward to remain valid.

## Acceptance examples

These are conditional behavioral examples, not executed tests. Unless stated otherwise, dependencies return normally, arithmetic and accesses are valid, regions do not overlap, and state remains stable except for the dependency effects explicitly described. Successful examples also require a representable signed increment.

### First available entry

The candidate is nonzero, the length is three, and the table contains a different nonzero value followed by two zero entries. Offset 104 contains one.

The optional dependency is skipped. The result is one. Offset 104 is written as two before the candidate is placed at index one. Index two remains untouched by placement.

### Index zero is a valid success

The length is one, the sole entry is zero, and offset 104 is zero.

There is no duplicate, and the unequal fields skip the optional dependency. Placement returns zero, after writing one to offset 104 and then placing the candidate in the sole entry.

### Duplicate beyond an empty entry

The length is three, and the entries are zero, the candidate, and zero.

The result is −1. The earlier empty entry is not used. Construction-related writes have occurred, but the optional dependency, direct increment, and candidate placement do not occur.

### No available entry and no implicit append

The length is two, both entries are nonzero and differ from the candidate, and offset 104 differs from two.

The result is −1. There is no direct extension of the search range and no placement beyond it.

### Dependency changes the later range

Before intervention, both compared fields contain two and duplicate detection finds no match. F-e901f09a5170 leaves a valid table with length four, offset 104 equal to two, and the first zero entry at index two.

The later search uses length four. The result is two, offset 104 becomes three before placement, and index two receives the candidate.

### Dependency introduces a duplicate

Duplicate detection initially finds no match, and the equality condition invokes F-e901f09a5170. That dependency leaves the candidate at index zero and a zero entry at index one within the later search range.

The function can still place the candidate at index one. No repeated duplicate check rejects this state.

### Nonpositive length

With length zero and offset 104 equal to one, both searches inspect no entries, the optional dependency is skipped, and the result is −1.

With both fields zero, the optional dependency is invoked. A resulting positive length and available slot can permit success; a resulting nonpositive length yields −1. Equal negative fields likewise permit the optional call.

### Zero candidate does not protect the write

Under nonwrapping arithmetic, a construction result of four derives a zero candidate and a second-input destination at address twelve.

If the write to that destination is valid and completes, the result is −1 without associated-record lookup. If the write is invalid, no normal return is guaranteed.

### Selected entry aliases the incremented field

The captured length is one. The table base points to record offset 104, whose value is zero. The candidate is nonzero.

The single entry is available, and the optional dependency is skipped. The function writes one to offset 104, then overwrites that same location with the candidate. The result is zero; the final field value is the candidate, not necessarily one.

## Uncertainties

- **documented_only — Dependency contracts:** Allocation, construction, lookup, failure signaling, preservation, ownership, and release behavior remain unavailable.
- **documented_only — Address model:** Four-byte address-like quantities are used, but byte order, alignment requirements, accessible ranges, and wrap behavior are not established.
- **documented_only — Arithmetic extremes:** No guaranteed wrap, trap, or clean-error behavior is established for invalid address arithmetic or signed overflow. In particular, incrementing the maximum signed 32-bit value has no specified successful outcome.
- **documented_only — Field semantics:** Search bounds and update behavior are established; capacity and occupancy invariants are not.
- **documented_only — Initializer meaning:** Its observed value is copied, but its type-level or domain-level meaning is unknown.
- **documented_only — Exceptional execution:** Dependency nonreturn, invalid accesses, and failures between writes have no established recovery contract.
- **documented_only — External mutation:** No synchronization, volatile-access, or concurrent-update semantics are established.
- **Inferred intent only:** The initial write may initialize a header, and the optional dependency may manage table storage. Neither interpretation is required by this contract.

## Evidence

The supplied static evidence establishes the two four-byte inputs, signed result, four dependency interactions, construction-relative accesses, zero-candidate rejection, record offsets, four-byte entries, ascending searches, differing length-observation rules, and increment-before-placement ordering.

The contract states those externally relevant relationships without assigning unavailable meanings to dependencies, storage ownership, or record fields. Acceptance examples are consequences conditional on their stated memory and arithmetic assumptions.

No runtime tests, execution traces, or dependency implementations have been examined.
