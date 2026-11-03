<!-- Trusted provenance: F-00a28bde1b8b; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-00a28bde1b8b

## Purpose

F-00a28bde1b8b conditionally obtains a reference associated with a caller-supplied record. It either:

- Creates and initializes a 16-byte region when the record’s reference and discriminator are both zero.
- Returns the record’s existing nonzero reference when its discriminator matches a global discriminator value.
- Rejects the record’s state with status 6.

The apparent intent is lazy creation and subsequent retrieval of discriminator-qualified state. That interpretation is inferred; the supplied fragment establishes the field operations and status values, but not the surrounding object model.

The routine is not an established safe allocation wrapper. On its creation path, it writes through the dependency’s result before testing whether the published output reference is zero. A null dependency result therefore does not establish a safely handled allocation failure.

## Interface

### Inputs

The routine accepts two address-bearing arguments:

1. **Record address.** Identifies storage containing at least these fields:

   | Offset from record address | Width | Role |
   |---|---:|---|
   | 0x14 | 32 bits | Stored reference |
   | 0x18 | 16 bits | Stored discriminator |

2. **Output-slot address.** Identifies writable storage for a 32-bit reference value.

These names describe observed use, not declared ownership or semantic types.

The fragment represents the record address and stored references using 32-bit integer-sized values. Its address handling is consistent with a 32-bit address model; portability to wider pointers is not established. The output slot receives four bytes, not an unspecified native-width pointer.

### Other input state

A global discriminator is represented as a 32-bit unsigned value. The routine evaluates it at entry and retains that value for possible use when creating new state. The existing-reference path also evaluates the global discriminator when checking the record.

The 16-bit record discriminator is interpreted as unsigned for comparison. On creation, only the low 16 bits of the entry-time global value are stored.

### Result

The normal return value is a 32-bit status:

| Status | Established meaning |
|---:|---|
| 0 | A creation or existing-reference path reached its success return |
| 6 | The record did not satisfy either accepted state |
| 28, hexadecimal 0x1c | The creation path reached its post-publication check and observed a zero output-slot value |

Status 28 is not evidence of a conventional allocation-failure protocol. Several writes precede that check.

### Preconditions and storage requirements

For defined, ordinarily usable execution:

- The record fields must be readable at their stated widths.
- The output slot must be writable whenever an accepted path uses it.
- The record reference field must be writable on the creation path.
- The record discriminator field must be writable if creation reaches its success update.
- The dependency result must support four consecutive 32-bit writes on the creation path.

The routine contains no explicit validation of these requirements. Required alignment, byte order, address-wrap behavior, and the platform’s response to invalid memory accesses are not established by the fragment.

No non-aliasing requirement is explicitly enforced. Nevertheless, aliasing among the output slot, record, returned region, and global storage can affect later reads and final state. Ordinary examples below assume those regions do not overlap.

## Behavior

### Entry observation and state selection

The routine first evaluates the 32-bit global discriminator and retains its value. This occurs even on paths that ultimately return status 6.

It then reads the record’s 32-bit reference field.

- If that field is zero, it examines the 16-bit record discriminator and enters the creation path only if that discriminator is also zero.
- If that field is nonzero, it examines the discriminator and follows the existing-reference comparison path.

The field widths matter independently: a zero reference with any nonzero 16-bit discriminator is rejected, regardless of the global discriminator.

### Creation path

Creation is selected only after observing both:

- A zero reference at record offset 0x14.
- A zero discriminator at record offset 0x18.

The routine then calls F-8157a180d901 with the numeric argument 16.

After that call returns, the caller-visible operations occur in this order:

1. The dependency’s returned address-sized value is written into the 32-bit output slot.
2. Four 32-bit zero values are written through the returned address, at offsets 0, 4, 8, and 12, in that order.
3. The output slot is read, and its current 32-bit value is written into the record’s reference field.
4. The output slot is read again and tested for zero.
5. If that read is nonzero, the low 16 bits of the entry-time global discriminator are written into the record’s discriminator field, and the routine returns status 0.
6. If that read is zero, the routine returns status 28 without performing the final discriminator write.

The initialization covers exactly 16 bytes through four word-sized stores. It is not evidence of initialization outside that range or of any richer internal structure.

The two reads of the output slot after initialization are behaviorally important. The routine does not simply retain the dependency’s result and use that retained value for publication and the final test. Aliasing or external changes can make the values read from the output slot differ from the original result.

The routine does not revalidate the record’s initial zero fields after the dependency call. Any effects of F-8157a180d901 on the record or other state are therefore relevant.

### Existing-reference path

When the initial reference-field read is nonzero, the routine compares the unsigned 16-bit record discriminator with the 32-bit global discriminator evaluated for this comparison.

This is a full numeric equality comparison, not a comparison against only the global value’s low 16 bits. Consequently, a global value greater than 65,535 cannot equal any possible record discriminator.

If the comparison matches:

1. The record’s 32-bit reference field is read again.
2. That newly read value is written to the output slot.
3. The routine returns status 0.

The reference copied to the output is thus a later field read, not necessarily the nonzero value that selected this path. The routine does not test the copied value for zero and does not validate or dereference it.

If the discriminator comparison does not match, the routine returns status 6 without an explicit output-slot write.

### Rejection paths

Status 6 is returned for either of these states:

- The initial reference is zero and the record discriminator is nonzero.
- The initial reference is nonzero and the discriminator does not numerically equal the global discriminator used for the comparison.

Neither rejection path invokes F-8157a180d901. Neither explicitly initializes or clears the output slot, and neither explicitly modifies the record.

### Discriminator-width asymmetry

Creation and retrieval use different width rules:

- Creation stores only the low 16 bits of the entry-time global discriminator.
- Retrieval compares the 16-bit stored discriminator against the full 32-bit global value.

For example, creating state while the global value is 65,537 stores discriminator 1. With that same global value on a later call, the stored discriminator cannot pass the existing-reference comparison.

This asymmetry is established by the represented operations. Whether it is intentional is unknown.

## Side effects

### Direct writes

On an ordinary successful creation path, the routine writes:

- Four bytes to the output slot.
- Sixteen bytes of zeros through the dependency result.
- Four bytes to the record reference field.
- Two bytes to the record discriminator field.

On a successful existing-reference path, its only explicit write is the four-byte output-slot write.

On status-6 paths, it performs no explicit writes.

On the status-28 path, the output publication, four initialization stores, and record-reference assignment have already occurred. There is no rollback.

### Publication and partial completion

The dependency result becomes visible in the output slot before the returned region is zeroed. The record reference is written only after those four initialization stores.

An invalid access or interrupted execution can therefore leave partially updated storage. The fragment provides no cleanup path, rollback mechanism, or exception handling.

No synchronization, atomic publication protocol, or memory-ordering guarantee is established. The stated order describes the represented operations, not a guarantee of safe concurrent observation.

### Ownership and lifetime

The fragment contains no release operation.

Creation appears to attach a newly obtained region to the record, but neither ownership transfer nor the region’s lifetime is established. In particular, the fragment does not establish:

- That F-8157a180d901 returns newly allocated storage.
- That the result is unique or initially uninitialized.
- That the caller may release the returned reference.
- That the record owns the region.
- Which dependency, if any, releases it.

The existing-reference path copies a reference; it does not establish a new allocation or a distinct ownership interest.

### Aliasing

Aliasing is not checked. It can materially change behavior because the output is published before initialization and subsequently read back.

For example, if the output slot overlaps a word that is zeroed in the returned region, initialization may overwrite the just-published reference. The later record assignment and zero test then operate on that overwritten value.

Likewise, overlap between the output slot and the record discriminator can make an output write alter the discriminator. No general claim that a successful call leaves all stored references and tags mutually consistent is valid without suitable storage separation.

## Dependencies

### F-8157a180d901

**Observed interface:** called with argument 16; its return value is used as a reference supporting four consecutive 32-bit stores.

**Invocation condition:** only after the record’s reference and discriminator have both been observed as zero.

**Unknown contract:** allocation semantics, failure signaling, alignment, ownership, lifetime, synchronization, and effects on other memory or global state are documented_only.

The routine itself does not check the returned value before dereferencing it. If the dependency returns zero and zero is not a writable address, the initialization attempt prevents any demonstrated normal return of status 28.

The entry-time discriminator is retained across this call for the eventual tag write. Changes the dependency makes to the global discriminator do not replace that retained value for this particular write. Other dependency effects must not be assumed absent.

## Acceptance examples

These are conditional static acceptance examples, not executed tests. Unless stated otherwise, assume valid, nonoverlapping storage, stable global and record state, and ordinary completion of all accesses.

1. **Create from an empty record.**  
   Given reference 0, discriminator 0, and entry-time global value 7, suppose F-8157a180d901 returns a valid nonzero address for 16 writable bytes. The output receives that address, the 16 bytes are zeroed, the record reference receives the output value, the record discriminator becomes 7, and the result is 0.

2. **Reject a tagged empty record.**  
   Given reference 0 and discriminator 7, the result is 6. The dependency is not called; the output and record receive no explicit writes.

3. **Retrieve a matching reference.**  
   Given a stable nonzero reference, discriminator 7, and comparison-time global value 7, the output receives that reference and the result is 0. The referenced region is not accessed.

4. **Reject a mismatch.**  
   Given a nonzero reference, discriminator 7, and comparison-time global value 8, the result is 6. The output is not explicitly changed.

5. **Zero is a valid matching discriminator.**  
   Given a nonzero reference, discriminator 0, and comparison-time global value 0, retrieval succeeds. A zero discriminator alone does not imply an empty record.

6. **Truncation does not imply later equality.**  
   Creation under global value 65,537 stores discriminator 1. A later retrieval under global value 65,537 returns 6, assuming the reference remains nonzero.

7. **A null dependency result is not a safe-failure example.**  
   If the dependency returns zero, the routine first publishes zero and then attempts a 32-bit zero store through address zero. Normal return of 28 is not established without additional memory-access assumptions.

8. **Aliasing can reach the zero-output branch.**  
   If the output slot overlaps a word in the valid returned region, an initialization store can zero the output. If subsequent operations complete and the output remains zero, the record reference becomes zero, the routine returns 28, and it omits the final discriminator write.

## Uncertainties

The following remain documented_only:

- The record’s semantic type and the meaning of its discriminator.
- The source, mutability, and permitted range of the global discriminator.
- The complete contract of F-8157a180d901.
- Any caller-enforced validity, alignment, or non-aliasing requirements.
- Actual concurrency behavior and platform-specific memory-access semantics.
- Whether status 28 is reachable in intended use.
- Whether discriminator truncation is deliberate or erroneous.
- Whether the two represented global evaluations correspond to distinct physical memory reads.

Lazy allocation and discriminator-based compatibility checking are inferred intent, not independently established guarantees.

## Evidence

The supplied fragment supports the field offsets, access widths, branch conditions, dependency argument, ordered initialization stores, output rereads, discriminator truncation, and return values described above.

Its names and surrounding implications are treated as untrusted data. This contract uses the fragment as static evidence of represented behavior, not as proof of dependency semantics or successful execution.

No implementation access, runtime execution, or tests have been performed. The acceptance examples are conditional deductions from the supplied operations.
