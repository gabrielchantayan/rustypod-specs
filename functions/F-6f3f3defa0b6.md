<!-- Trusted provenance: F-6f3f3defa0b6; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-6f3f3defa0b6

## Purpose

F-6f3f3defa0b6 searches a packed range table and computes a mapping. A context flag enables special selection among neighboring ranges.

Two modes are exposed:

- **Lookup:** search using the supplied unsigned 32-bit key and return a mapping without direct state writes.
- **Advancing:** search using the supplied key plus one, modulo \(2^{32}\), then conditionally call dependencies and publish key and mapping state.

Zero is the common absence or failure result. It can represent an empty table, an unsuccessful search, an excluded range, a zero mapping entry, arithmetic reduction to zero, advancing-mode termination, or dependency-mediated failure. The return value does not distinguish these causes.

“Range mapping” describes the established structure. A particular character encoding, resource format, or application-level meaning is not established.

## Interface

### Arguments and result

| Input | Contract |
|---|---|
| Context address | Address of accessible context storage containing the fields below. |
| Key address | Address of a readable native unsigned 32-bit value; successful advancing paths also require writable storage. |
| Mode | A 32-bit integer: zero selects lookup; every nonzero value selects advancing. |

The result is unsigned 32-bit. Locally calculated mappings are reduced to 16 bits. A mapping obtained from context after a dependency call is returned at its full 32-bit width.

### Context layout

Offsets are bytes from the context address.

| Offset | Width | Meaning |
|---|---:|---|
| `0x10` | 32-bit address | Table address captured at entry |
| `0x14` | 32 bits | Flags; mask `0x2` enables overlap-aware selection |
| `0x18` | 32 bits | Key state used during advancing completion |
| `0x1C` | 32 bits | Mapping state used during advancing completion |

These fields and caller key storage use native representation. Table quantities are assembled explicitly as unsigned big-endian 16-bit values.

### Table layout

Let **B** be the captured table address. Let **H** be the unsigned big-endian 16-bit quantity at `B + 6`, **S** be H with its low bit cleared, and **N = S / 2**.

S ranges from 0 to 65,534 and N from 0 to 32,767. H values zero and one both yield an empty table. The low bit of H affects neither count nor array spacing.

For index **i**, where \(0 \le i < N\):

| Field | Address |
|---|---|
| Upper bound | `B + 14 + 2i` |
| Lower bound | `B + 16 + S + 2i` |
| Adjustment | `B + 16 + 2S + 2i` |
| Relative displacement | `B + 16 + 3S + 2i` |

Every field is an unsigned big-endian 16-bit quantity. Adjustments are not sign-extended; their effect is modular addition.

A nonzero displacement is measured in bytes from the address of its own displacement field, not from B.

### External global

The function reads one external global at two distinct program points:

1. On every nonempty-table path, after preparing the effective key and before searching, to initialize the retained upper-bound state.
2. Only after finding a containing range with overlap-aware selection disabled, to obtain the value compared with that range’s displacement.

Call the values observed **K-entry** and **K-compare**. They are separate reads, not one captured value. Their numeric values and equality are not established. The supplied representation uses them in unsigned 32-bit state and comparisons; the external object’s declaration and physical storage width are not supplied.

### Preconditions and ownership

There are no local pointer, allocation-length, alignment, range-consistency, or displaced-index checks. All storage actually accessed must remain valid.

Useful binary-search behavior requires appropriate table ordering. Stronger claims that all overlap candidates contain the key require additional table invariants. Neither is checked locally.

No allocation, release, or ownership transfer is directly visible. Dependency ownership and lifetime effects remain unknown.

## Behavior

### Entry and empty-table exit

The function reads, in order:

1. The caller’s key.
2. The context table address.
3. The header quantity at `B + 6`.

If S is zero, it returns zero immediately. It does not read the external global or flags, call either dependency, or directly write context or caller storage.

For a nonempty table, the **effective key** is the original key in lookup mode and the original key plus one modulo \(2^{32}\) in advancing mode.

It then performs the K-entry global read before entering the search. This initializes retained upper-bound state even though a nonempty search subsequently reads a table upper bound before normal completion. The global read remains part of the access behavior.

The effective key remains unsigned 32-bit; it is not narrowed to the 16-bit endpoint width.

### Initial search

The function uses a midpoint search over N entries. At each examined index, it reads the upper bound and then the lower bound.

Comparisons are unsigned:

- A lower bound greater than the effective key directs the search toward lower indices.
- Otherwise, an upper bound below the effective key directs it toward higher indices.
- Otherwise, the interval inclusively contains the effective key.

For a containing interval, the next reads are its adjustment, its displacement, and then context flags.

An unsuccessful search leaves the provisional mapping zero. It does not read flags or perform the K-compare read.

### Ordinary selection

With flag mask `0x2` clear, the function performs the separate K-compare global read after reading the selected entry’s adjustment, displacement, and flags.

If the displacement equals K-compare, mapping calculation is skipped and the provisional result stays zero. Otherwise, the initially selected entry is mapped.

There is no established equality between K-compare and `0xFFFF`. In particular, K-compare is used in a 32-bit comparison against a zero-extended 16-bit displacement; a K-compare value above 65,535 cannot equal any displacement.

### Overlap-aware selection

With flag mask `0x2` set, the second global read is not performed. Candidate qualification instead uses the literal displacement marker `0xFFFF`.

The initially found entry is a candidate if its displacement is not `0xFFFF`.

#### Backward examination

The function examines preceding entries in decreasing index order, continuing while each preceding upper bound is at least the effective key.

For each entry admitted by that upper-bound check, field reads occur in this order:

1. Upper bound.
2. Lower bound.
3. Adjustment.
4. Displacement.

Every nonmarker displacement replaces the candidate. Thus, if backward examination finds qualifying entries, the lowest-index qualifying entry in that run wins.

There is no lower-bound-versus-key check during this examination. Candidate membership in an interval therefore depends on table invariants beyond the checks actually made.

If a preceding upper bound is below the key, that upper bound is read, but the entry’s other fields are not read. This rejected probe does not replace the retained working entry.

#### Forward fallback and its exceptional restoration order

Forward examination occurs only if the initial entry has displacement `0xFFFF` and backward examination found no nonmarker candidate.

Before proceeding forward, the function resets the working index to the initially found index. If backward examination had moved away from that index, it rereads the initial entry in this specific order:

1. Adjustment.
2. Displacement.
3. Upper bound.
4. Lower bound.

This restoration is not bounds-first. It differs from both initial selection and the later restoration of a chosen candidate. If backward examination never moved away, these restoration reads are omitted.

The function then examines subsequent entries in increasing index order:

- Read upper bound, then lower bound.
- Stop if the lower bound exceeds the effective key.
- Otherwise, read adjustment, then displacement, and retain that entry as working state.
- Replace the candidate whenever its displacement is not `0xFFFF`.

Forward examination does not stop at the first candidate. The highest-index qualifying entry in the admitted forward run wins.

There is no check that an admitted forward entry’s upper bound reaches the key. A stopping probe reads both bounds but does not replace retained working state.

If no candidate exists, mapping is skipped. Otherwise, if the chosen candidate differs from the last working entry, its fields are reread in this order: upper bound, lower bound, adjustment, displacement. Mapping then uses the chosen index and fields.

The directional preference is asymmetric: backward selection prefers the earliest qualifying entry; fallback forward selection prefers the latest.

### Mapping calculation

For the chosen entry:

- Displacement zero uses the effective key as the mapping base.
- Nonzero displacement reads a big-endian 16-bit mapping entry at:

\[
\text{displacement-field address}
+\text{displacement}
+2(\text{effective key}-\text{lower bound})
\]

The key subtraction and multiplication use unsigned 32-bit arithmetic, including wraparound. The function does not validate the resulting index or address.

A displaced mapping entry of zero suppresses adjustment and leaves the result zero.

Otherwise, the provisional mapping is:

\[
(\text{mapping base}+\text{adjustment}) \bmod 65{,}536.
\]

The result can become zero through this final reduction. Direct mappings also undergo the same reduction.

### Lookup completion

Lookup returns the provisional result. It calls neither dependency and performs no direct writes.

### Advancing termination

Before calling dependencies, advancing mode returns zero if both conditions hold:

- The retained upper bound is below the effective key.
- The retained working index plus one equals N.

This is a test of retained search state, not a separately calculated maximum endpoint.

After an unsuccessful binary search, that state identifies the last examined entry. Mapping calculation retains the chosen entry. An overlap path that finds no candidate can retain the last fully examined entry even though no mapping was calculated. Rejected neighbor probes do not themselves become retained state.

This early exit performs no direct writes and calls neither dependency.

### Advancing completion through dependencies

Paths surviving termination call **F-0200cac86fa9** once with the context address.

If it returns zero:

1. Write the effective key to context `0x18`.
2. If the provisional mapping is nonzero, write it to context `0x1C`.
3. Otherwise, call **F-26bd5ca214f5**, then read context `0x1C` as a full 32-bit value.
4. If that read is zero, return zero without writing the caller’s key.
5. On success, read context `0x18`, write that value to caller key storage, and return the mapping.

The published key is the value from the final context read. It is not necessarily the effective key: the second dependency may have changed it.

If F-0200cac86fa9 returns nonzero:

- A nonzero provisional mapping causes publication of the effective key directly to caller key storage, followed by return of that mapping.
- A zero provisional mapping returns zero without publication.

This branch performs no direct context writes and does not call F-26bd5ca214f5. It does not reverse effects of the first dependency.

## Side effects

Direct writes occur only during advancing completion:

- Context `0x18` after a zero return from F-0200cac86fa9.
- Context `0x1C` on that branch when a nonzero provisional mapping already exists.
- Caller key storage on successful publication.

The table and external global are only directly read. Dependencies may modify them; no preservation guarantee is established.

Aliasing is neither prohibited nor checked. If caller key storage aliases context `0x1C`, final key publication can overwrite mapping state while the returned mapping remains the already retained value. Claims about final independent key and mapping fields require appropriate nonaliasing.

The table address is captured before search. Both possible global reads and all table reads occur before dependency calls. No later table or external-global reads are visible.

Big-endian assembly establishes each two-byte value’s interpretation, **not the sequencing of its two constituent byte accesses**. The supplied representation does not establish high-byte-first or low-byte-first evaluation. Field-level ordering described above does not imply atomic reads, synchronization, or a consistent snapshot under concurrent mutation.

## Dependencies

| Opaque ID | Established interaction | Unresolved behavior |
|---|---|---|
| F-0200cac86fa9 | Receives the context address once on advancing paths surviving termination. Zero versus nonzero selects the completion branch. | Meaning of status, context/global mutations, ownership, lifetime effects, and failure mechanisms. |
| F-26bd5ca214f5 | Receives the context address only when the first dependency returned zero and the provisional mapping is zero. Its return value is not used; context `0x1C` is read afterward. | Mapping production, changes to key state, other mutations, ownership, lifetime effects, and failures. |

Neither dependency is assumed to preserve globals, unrelated fields, or the validity of subsequently accessed memory.

## Acceptance examples

These are conditional static acceptance cases, not executed tests. Storage is valid and nonaliasing unless specified otherwise.

1. **Empty table:** With H equal to zero or one, return zero in either mode. Perform neither external-global read, read no flags, call neither dependency, and make no direct writes.

2. **Distinct global-read occurrences:** On a nonempty lookup that finds no interval, perform the entry global read but not the comparison read. For an ordinary containing interval, perform both reads, with the second after adjustment, displacement, and flags. On an overlap-aware containing interval, perform only the entry global read.

3. **Independent comparison observation:** Conditional on K-entry being 8 and K-compare being 12, an ordinary selected displacement of 12 suppresses mapping. The earlier observation of 8 does not determine this decision. This example does not establish a mechanism that changes the global between reads.

4. **Inclusive direct mapping:** For bounds 10 through 20, adjustment 3, displacement zero, and ordinary selection with K-compare unequal to zero, lookup key 20 returns 23.

5. **Displaced zero suppression:** For lower bound 100, adjustment 7, displacement 8, and key 102, read 12 bytes beyond the displacement field. Subject to selection permitting that displacement, entry 9 returns 16; entry zero returns zero rather than 7.

6. **Modular zero:** A permitted direct mapping of key 1 with adjustment 65,535 returns zero.

7. **Different exclusion rules:** Ordinary selection excludes displacement equal to K-compare. Overlap-aware selection uses literal `0xFFFF` instead. If K-compare exceeds 65,535, ordinary selection cannot exclude any displacement by equality.

8. **Backward preference:** If two preceding entries pass backward examination and both have nonmarker displacements, select the lower-index one.

9. **Fallback restoration order:** If the initial displacement is `0xFFFF`, backward examination moves away but finds no candidate, reread the initial entry as adjustment, displacement, upper bound, lower bound before forward examination.

10. **Forward preference and final restoration:** If fallback admits two nonmarker entries, the later wins. If a subsequent admitted marker entry becomes working state, restore the chosen entry as upper bound, lower bound, adjustment, displacement before mapping.

11. **Increment wraparound:** Advancing from caller key `0xFFFFFFFF` searches with effective key zero, without saturation or rejection.

12. **Dependency-mediated full-width result:** If the provisional result is zero, termination does not apply, the first dependency returns zero, and the second leaves key state 123 and mapping state `0x12345678`, publish 123 and return `0x12345678` without 16-bit reduction.

13. **Failure after mutation:** On that dependency path, mapping state zero causes return zero without caller-key publication. The earlier context key write and dependency effects remain observable.

## Uncertainties

- **documented_only — External global:** Its definition, physical width, numeric value, mutability, and relationship between the two observed values are unavailable. Neither observation is established as `0xFFFF`.
- **documented_only — Dependency semantics:** Preservation, ownership, lifetime, validation, and recovery guarantees are unavailable.
- **documented_only — Table invariants:** Ordering and overlap constraints sufficient for intended interval semantics are inferred requirements, not enforced conditions.
- **documented_only — Invalid memory:** No defined local error result covers inaccessible storage, displaced out-of-bounds reads, or dependency-invalidated addresses.
- **documented_only — Access sequencing:** Byte order within each two-byte read, atomicity, concurrency behavior, and snapshot consistency are not established.
- **documented_only — Domain intent:** No specific encoding, resource type, or meaning of mapping zero beyond the observed branches is established.

## Evidence

Established static evidence is the supplied function representation and opaque dependency mapping. It supports the context offsets, packed-table addresses, unsigned comparisons, arithmetic widths, two distinct external-global read sites, field-level read ordering, candidate selection, direct write ordering, dependency conditions, and return paths.

The represented execution model uses 32-bit integers and addresses, with explicitly assembled 16-bit table quantities. Broader portability and invalid-address semantics are not established.

Intent is identified separately from mechanics. No runtime tests have been performed.
