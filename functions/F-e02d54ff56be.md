<!-- Trusted provenance: F-e02d54ff56be; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-e02d54ff56be

## Purpose

F-e02d54ff56be searches a dependency-provided sequence for the first candidate satisfying two flag conditions. For that candidate, it computes a two-bit return value and writes seven pieces of associated information to caller-supplied destinations. It then finalizes the traversal.

If traversal ends without a qualifying candidate, the function returns zero and performs no direct writes to those destinations.

Zero is also a possible return value for a qualifying candidate. The return value alone therefore does not distinguish “no candidate found” from “candidate found with a zero-valued result.”

These behaviors are established by the supplied static evidence. The meaning of the records, flags, and traversal mode is not established.

## Interface

The function has nine inputs, identified here by their argument positions rather than implementation identifiers:

| Position | Role | Representation and use |
|---|---|---|
| 1 | Unused input | A 32-bit value. The function does not inspect or forward it. |
| 2 | Context address | An address represented by a 32-bit quantity. A four-byte field is read at byte offset 60 from this address. |
| 3 | Embedded-address destination | Destination for a 32-bit address value identifying byte offset 8 within the selected candidate. |
| 4 | First flag destination | Destination for one byte derived from bit 5 of a candidate byte. |
| 5 | Second flag destination | Destination for one byte derived from bit 6 of a candidate byte. |
| 6 | Third flag destination | Destination for one byte derived from bit 7 of a candidate byte. |
| 7 | Fourth flag destination | Destination for one byte derived from bit 0 of the following candidate byte. |
| 8 | First payload destination | Destination for an uninterpreted four-byte candidate field. |
| 9 | Second payload destination | Destination for another uninterpreted four-byte candidate field. |

The return value is a 32-bit unsigned quantity. On ordinary completion, it is either zero because traversal exhausted, or a value from zero through three computed from a selected candidate.

### Required accessible storage

The context field at offset 60 must be readable for four bytes before traversal initialization.

For each candidate reported by the advancement dependency, the function must be able to read its byte at offset 24. A selected candidate additionally requires:

- A four-byte read beginning at offset 24.
- A byte read at offset 25.
- Four-byte reads at offsets 28 and 32.

On the successful-selection path, the third, eighth, and ninth arguments must identify writable four-byte destinations. Arguments four through seven must identify writable one-byte destinations.

The function performs no explicit null checks, range checks, alignment checks, or destination-capacity checks. The supplied evidence does not establish what happens when these access requirements are violated.

The seven destinations need not be distinct according to any explicit check. However, overlap can change final contents and, where destinations overlap candidate storage, can change values read later in the operation.

## Behavior

### Initialization

The function first establishes a tentative return value of zero.

It reads the four-byte context field at offset 60 and supplies that value to F-538044b67754 together with:

- The address of a private 20-byte traversal-state region.
- The 32-bit bit pattern `0xFFFFFFFE`.

The bit pattern is established; whether the dependency interprets it as a negative number, a sentinel, a mask, or something else is unknown. It must not be described as a particular semantic traversal mode without additional evidence.

The initialization dependency’s return value is not used. No initialization-failure branch is visible.

### Advancing the traversal

After initialization, the function calls F-6e4651b532dc with the private traversal state and the address of a private four-byte candidate-address slot.

A zero return from this dependency ends the search. In that case, the function does not inspect the candidate-address slot for that advancement and proceeds to finalization with the tentative return value unchanged.

Any nonzero return is treated as indicating that the candidate-address slot can be used. Negative values, if representable by the dependency’s result, are not handled as a separate error class: they follow the same path as other nonzero values.

The function contains no independent traversal bound. Continued nonzero advancement results and continued rejection of candidates can cause arbitrarily many iterations.

### Candidate selection

For each reported candidate, selection uses two separate byte reads from offset 24:

1. The first read supplies the value whose bit 4 may qualify the candidate.
2. The second read supplies the value whose bit 0 determines whether the candidate is immediately rejected.

A candidate is rejected when bit 0 of the second read is clear. If that bit is set, the candidate is accepted only when bit 4 of the first read is also set.

For stable, ordinary memory, this reduces to requiring both bit 0 and bit 4 of the byte at offset 24 to be set. Other bits do not affect selection.

The two reads must nevertheless remain distinct in the behavioral contract. If the underlying byte can change between reads, selection combines bit 4 from the first observation with bit 0 from the second. There is no established atomic snapshot.

Rejected candidates cause another advancement call. Their output fields are not copied to caller destinations.

The first candidate accepted by this process ends the search. There is no subsequent advancement after selection.

### Extracting the result and outputs

Once a candidate is selected, the function performs the following direct accesses in order:

| Order | Access or effect |
|---|---|
| 1 | Read four bytes beginning at candidate offset 24; retain bits 1 and 2, shifted down to produce the return value. |
| 2 | Write the address of candidate offset 8 to the third argument’s four-byte destination. |
| 3 | Read the byte at candidate offset 24; write its bit 5 as a byte-valued zero or one to the fourth argument’s destination. |
| 4 | Read the byte at candidate offset 24 again; write its bit 6 as a byte-valued zero or one to the fifth argument’s destination. |
| 5 | Read the byte at candidate offset 24 again; write its bit 7 as a byte-valued zero or one to the sixth argument’s destination. |
| 6 | Read the byte at candidate offset 25; write its bit 0 as a byte-valued zero or one to the seventh argument’s destination. |
| 7 | Read four bytes at candidate offset 28; copy that four-byte value to the eighth argument’s destination. |
| 8 | Read four bytes at candidate offset 32; copy that four-byte value to the ninth argument’s destination. |

The return calculation uses a four-byte read, not the earlier selection byte. Its numeric operation is to mask the read value with seven and shift right by one. Its range is therefore zero through three.

Relating those two result bits to particular bits of the lowest-addressed byte requires a byte-order assumption. On a little-endian target, they are bits 1 and 2 of the byte at offset 24. The supplied evidence does not independently establish target byte order.

Each one-byte flag output is normalized to zero or one. The two four-byte payload outputs are not interpreted, validated, or normalized.

The embedded-address output is computed rather than loaded from an address field within the candidate. The function does not dereference the resulting address at offset 8.

The repeated reads are significant. Selection, return calculation, and flag extraction need not observe the same contents if candidate storage changes or an earlier output write overlaps it.

### Finalization and return

Both normal termination paths—exhaustion and selection—call F-944b595e25e9 with the private traversal state.

The function then returns the retained result. It does not use the finalization dependency’s return value.

This establishes finalization on the two visible normal paths. It does not establish finalization after an invalid memory access, a dependency that fails to return, or another abnormal transfer of control.

## Side effects

The function directly writes caller destinations only after selecting a candidate. Those writes occur in the order listed above and are not transactional.

In particular:

- If destinations overlap one another, later writes can overwrite earlier outputs.
- If a destination overlaps candidate storage, its write can affect subsequent reads.
- The return calculation precedes all seven direct output writes, so those writes do not cause the function to recompute that result.
- A failure during output processing can leave only a prefix of the writes completed; no rollback is visible.

On exhaustion, the function makes no direct writes through arguments three through nine. This is narrower than a guarantee that their contents remain unchanged: called dependencies may have other access to the same memory.

The function exposes an address into the selected candidate rather than making a copy of the embedded region. No ownership transfer, allocation, reference-count increment, or lifetime extension is visible.

Finalization occurs after that address has been written. Whether the address remains valid after finalization depends on dependency behavior and candidate ownership, neither of which is established.

The function does not directly update the context field it initially reads. Indirect changes by dependencies remain possible.

## Dependencies

| Opaque function ID | Established use | Unestablished behavior |
|---|---|---|
| F-538044b67754 | Called once before traversal, with private state, the context’s four-byte field, and `0xFFFFFFFE`. Its result is ignored. | Initialization semantics, allocation, failure reporting, global changes, and ownership effects. |
| F-6e4651b532dc | Called repeatedly with private state and a candidate-address destination. Zero ends traversal; every nonzero result causes candidate inspection. | Candidate ordering, mutation, storage lifetime, error conventions, and any guarantee of eventual termination. |
| F-944b595e25e9 | Called once on each visible normal termination path, after any selected-candidate output writes. Its result is ignored. | Resource release, candidate invalidation, global changes, and effects on memory accessible through other means. |

No dependency is assumed to preserve globals or to allocate, retain, or release memory merely because of its position in this sequence.

## Acceptance examples

The following examples are conditional static expectations, not reported test executions. Unless stated otherwise, memory is valid and stable, outputs are disjoint from candidate storage and one another, and dependencies return normally without independently modifying the observed output destinations.

### Empty traversal

If the first advancement returns zero:

- Initialization has already occurred.
- No candidate fields are read by this function.
- No direct output writes occur.
- Finalization occurs.
- The return value is zero.

### Rejection before selection

If successive candidates have offset-24 bytes `0x01`, `0x10`, and `0x11`, respectively:

- The first is rejected because bit 4 is clear.
- The second is rejected because bit 0 is clear.
- The third is selected.
- No fourth advancement occurs.
- Outputs come from the third candidate.

This example assumes each candidate’s two selection reads observe the stated byte.

### Selected candidate with a zero result

On a little-endian target, suppose a selected candidate has:

- `0x11` at offset 24.
- An even byte at offset 25.

The return value is zero. All four byte-valued flag outputs are zero. The embedded address and both four-byte payload values are still written.

This differs from exhaustion even though both paths return zero.

### Selected candidate with all extracted flags set

On a little-endian target, suppose the byte at offset 24 is `0xF7` and the byte at offset 25 is `0x03`.

The candidate qualifies. The return value is three, and all four byte-valued flag outputs are one. The payload outputs equal the four-byte values read at offsets 28 and 32.

### Distinct selection observations

Suppose the first offset-24 read observes `0x10` and the second observes `0x01`.

The candidate qualifies: the first observation supplies a set bit 4, and the second supplies a set bit 0. Neither observation individually needs to contain both bits.

Later result and output reads use their own observations.

### Overlapping byte destinations

If arguments four and five identify the same byte, the bit-6-derived write occurs after the bit-5-derived write. Subject to later aliasing and dependency effects, the shared byte ends with the bit-6-derived value.

### Nonzero advancement status

If advancement returns a negative nonzero status and supplies a usable candidate address, the function inspects that candidate. It does not interpret that status as an error requiring immediate finalization.

## Uncertainties

The following are **documented_only**:

- The semantic meaning of the context field and candidate records.
- The interpretation of `0xFFFFFFFE`.
- Target byte order and its consequences for mapping the four-byte return calculation to individual addressed bytes.
- Alignment requirements and behavior of invalid or unaligned accesses.
- Address-overflow behavior when forming context and candidate offsets.
- Whether dependency calls allocate or release resources, modify globals, or invalidate candidates.
- The lifetime and ownership of the returned embedded address.
- Any distinction between advancement exhaustion and dependency-reported errors.
- Whether candidate memory can change asynchronously and whether such access is supported by the execution environment.
- Guarantees of termination, exception behavior, or recovery after faults.

Interpreting the dependencies as traversal initialization, advancement, and finalization is inferred intent supported by their usage pattern. Their internal contracts are not supplied.

## Evidence

This contract is based solely on the supplied static function body and opaque dependency mapping.

Established observations include argument usage, access widths and offsets, the two-read selection condition, result masking, output ordering, normal finalization paths, and ignored dependency results.

No dependency implementations, runtime traces, platform memory model, or independent layout definitions were supplied. No runtime tests have been performed.
