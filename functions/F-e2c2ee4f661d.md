<!-- Trusted provenance: F-e2c2ee4f661d; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-e2c2ee4f661d

## Purpose

F-e2c2ee4f661d searches a context-provided table in ascending index order and selects the first entry whose two stored values satisfy bucket-based comparisons against a processed request. A nonzero relaxation control permits a mismatch in the first comparison, but never in the second.

A context flag and a request discriminator gate processing. The function returns a 32-bit status and, on success, optionally writes the selected zero-based index to caller-provided storage.

“Dimension,” “normalization,” and “missing dimension” below describe inferred intent. The established behavior consists of field reads, conditional calls, arithmetic, substitution, ordered comparisons, and a success-only output write. The application domain and units are not established.

## Interface

### Arguments and result

| Argument | Role | Established representation |
|---|---|---|
| Context | Provides the enabling flag, signed entry count, and table address | Address carried in a 32-bit integer |
| Request | Provides a discriminator, two raw dimensions, and their factors | Address of a byte-addressed record |
| Relaxation control | Permits a first-dimension bucket mismatch | 32-bit integer; zero disables relaxation, any nonzero value enables it |
| Optional index destination | Receives the selected index on success | Null or an address writable for one signed 32-bit integer |

The return value is 32 bits. The four explicit return values are:

| Status | Established meaning |
|---:|---|
| 0 | A matching entry was selected |
| 7 | The request discriminator was nonzero |
| 23 (`0x17`) | The current index reached or exceeded the signed entry count |
| 35 (`0x23`) | The required context flag was absent |

These meanings do not establish a broader error taxonomy. No explicit status represents arithmetic overflow, invalid storage, or failure within the dependency.

### Context layout

Offsets are byte offsets from the context address.

| Offset | Width | Use |
|---|---:|---|
| +8 | 32 bits | Flag word; mask `0x00000002` must be set |
| +28 | 32 bits | Signed entry count |
| +32 | 32 bits | Address of the table’s first entry |

The function does not directly inspect other context fields.

### Request layout

| Offset | Width | Use |
|---|---:|---|
| +0 | 8 bits | Discriminator; must equal zero |
| +4 | 32 bits | Signed raw first dimension |
| +8 | 32 bits | Signed raw second dimension |
| +12 | 32 bits | Signed first-dimension factor |
| +16 | 32 bits | Signed second-dimension factor |

Only the first byte is checked as the discriminator. No use of bytes +1 through +3 is established.

There is no explicit validation of dimension or factor sign, magnitude, or application-specific range.

### Table-entry layout

Entries have a 16-byte stride.

| Entry-relative offset | Width | Use |
|---|---:|---|
| +8 | 32 bits | Signed stored first dimension |
| +12 | 32 bits | Signed stored second dimension |

The function does not directly read the first eight bytes of an entry.

### Storage and arithmetic preconditions

Every address actually dereferenced must identify accessible storage of the required width. A nonnull output destination must permit a 32-bit write if selection succeeds. The function does not validate these requirements.

There is no general null check for the context, request, or table. Early returns may avoid some accesses, but are not pointer-validation behavior.

The table base and entry addresses must remain valid for entries actually examined. Signed multiplication and addition used to construct addresses also need to be representable for an unambiguous contract under the supplied representation. No portable wrapping policy is established for those signed expressions.

## Behavior

### Context gate and error precedence

The first direct read is the 32-bit context flag at +8. If mask `0x00000002` is clear, the function returns 35 immediately.

On this path, it does not directly read the request, table count, table address, or output destination. Other flag bits do not affect this decision.

When the required flag is present, the function reads the request discriminator byte at +0. A nonzero byte produces return status 7. No normalization call or table traversal occurs, and there is no direct output write.

Consequently, a missing context flag takes precedence over an invalid discriminator.

### First-dimension processing

The function reads the first-dimension factor at request offset +12 to distinguish two cases:

- If the factor is zero, it reads the raw first dimension at +4 and uses that value directly.
- If the factor is nonzero, it calls F-870c4fcf7295. The first argument is the raw first dimension multiplied by the factor, then increased by 36. The second argument is 72. The returned 32-bit value becomes the working first dimension.

The nonzero branch contains a further factor read for the multiplication. The factor tested for zero is therefore not established as a saved snapshot used by the arithmetic. Under stable storage, both reads have the same value.

The order between the raw-dimension and factor reads within the multiplication expression is not established.

### Second-dimension processing

After first-dimension processing completes, the function applies the same rules to the second dimension:

- The factor is at +16.
- The raw dimension is at +8.
- A zero factor selects the raw dimension directly.
- A nonzero factor causes a call to F-870c4fcf7295 with the dimension-specific product plus 36, and with 72 as the second argument.

Here too, the nonzero branch reads the factor again for the multiplication.

There are at most two direct dependency calls. If both occur, the first-dimension call completes before the second factor is tested. A first call may therefore affect the state observed during second-dimension processing if the dependency can modify shared storage.

No dependency return is classified as an error by this caller. Every returned 32-bit value is consumed as a working dimension if the call returns normally.

A zero factor bypasses the dependency; it does not multiply the raw dimension by zero.

### Normalization arithmetic

The multiplication and subsequent addition of 36 are represented as signed 32-bit arithmetic.

When both intermediate operations are representable, the dependency argument is established exactly. For example, raw value 10 and factor 2 produce the first argument 56.

When either signed operation overflows, the supplied representation does not establish a portable result. Neither unsigned wraparound nor a particular rounding or saturation rule may be assumed.

The dependency’s own mathematical behavior is separately unknown.

### Raw-dimension substitution

After both working dimensions have been established, the function rereads the raw first dimension.

- If that reread is zero, it reads the raw second dimension. A nonzero second dimension causes the working first dimension to be replaced by the working second dimension.
- If the reread first dimension is nonzero, it reads the raw second dimension. A zero second dimension causes the working second dimension to be replaced by the working first dimension.

Thus, for stable request storage:

| Raw first dimension | Raw second dimension | Result |
|---|---|---|
| Zero | Nonzero | Copy working second dimension into working first dimension |
| Nonzero | Zero | Copy working first dimension into working second dimension |
| Zero | Zero | No substitution |
| Nonzero | Nonzero | No substitution |

These decisions use raw values, not normalized results. A working value of zero does not itself trigger substitution.

Both normalization stages precede substitution. A dependency call may therefore compute a result that is subsequently overwritten. In particular, a zero raw dimension with a nonzero factor still causes its normalization call before substitution.

The rereads can observe dependency-induced changes to the request. No stable-request guarantee is supplied.

### Bucket transformation

For comparisons, define **B(v)** as follows:

1. Interpret the 32-bit value as an unsigned bit pattern.
2. Add 32 modulo 2³².
3. Clear the lowest six bits using mask `0xFFFFFFC0`.

Equivalently:

**B(v) = (((unsigned 32-bit representation of v) + 32) modulo 2³²) AND `0xFFFFFFC0`.**

The addition in this transformation is unsigned 32-bit arithmetic. Its modular behavior is established and is distinct from the unresolved signed overflow in normalization and address construction.

Bucket equality is bit-pattern equality. It is not an absolute-distance comparison. Values one unit apart can occupy different buckets; values 63 units apart can share a bucket.

No positivity check precedes this transformation.

### Ordered traversal and changing context fields

The search index starts at zero.

Before examining each candidate, the function reads the signed count at context offset +28. If the count is less than or equal to the current index, the function returns 23.

A zero or negative initial count therefore causes no table-entry reads. Normalization and substitution have already occurred.

When the count permits the candidate, the function reads the table base at context offset +32 and derives the entry address using a 16-byte stride. Both count and base are reread for each candidate; neither is held as a single table-wide snapshot.

For each candidate:

1. Read its second dimension at entry offset +12 and compare its bucket with the working second-dimension bucket.
2. If that comparison fails, reject the candidate without reading its first-dimension field.
3. If that comparison succeeds, read its first dimension at +8 and compare its bucket with the working first-dimension bucket.
4. Accept the candidate if the first-dimension comparison succeeds or the relaxation control is nonzero.

The first-dimension comparison is evaluated before the relaxation alternative. Relaxation therefore does not eliminate the first-dimension field read when the second comparison succeeds.

A rejected candidate advances the index by one. The first accepted candidate terminates traversal; no later candidate is inspected.

### Index arithmetic is bounded

Index increment does not present an unresolved signed-overflow case.

Each executed increment follows a count check that established:

**0 ≤ index < count ≤ 2,147,483,647.**

Therefore, the pre-increment index is at most 2,147,483,646, and the incremented value is at most 2,147,483,647. Both are representable as signed 32-bit integers.

Rereading a changing count does not invalidate this argument: the count read permitting that particular iteration already bounds its index. If the index reaches 2,147,483,647, no signed 32-bit count can permit another candidate or increment.

This bound does not establish safe entry-address arithmetic. Multiplication by the 16-byte stride or addition to the table base can exceed the signed range at much smaller indices.

### Successful completion

After a match:

- A nonnull output destination receives the selected index as one 32-bit integer.
- A null output destination suppresses that write.
- The function returns zero.

No direct output write occurs on any nonzero-status path.

## Side effects

The function’s only explicit memory write is the optional successful index write.

If the destination aliases the context, request, or table, the write modifies that aliased storage. No further search reads occur after this write.

The function itself contains no explicit allocation, release, pointer retention, or ownership transfer. This does not establish any corresponding guarantee for F-870c4fcf7295.

The dependency must not be assumed to preserve globals, leave input records unchanged, avoid allocation or release, or return values in a restricted range. Its effects can influence later reads, including:

- The second normalization stage.
- Raw-dimension substitution.
- The first count and table-base reads.
- Table contents examined during traversal.

The direct sequencing boundaries are:

1. Context flag read.
2. Request discriminator read.
3. First-dimension processing.
4. Second-dimension processing.
5. Raw-dimension rereads and substitution.
6. Repeated count checks, table-base reads, and candidate comparisons.
7. Optional successful output write.

Operand evaluation order within every arithmetic or equality expression is not established. No atomicity, synchronization, or consistent-snapshot guarantee is supplied.

“Unchanged output on failure” means no direct output write by this function. It is not a guarantee against dependency effects through aliases or shared state.

## Dependencies

### F-870c4fcf7295

This is the sole explicit function dependency.

For each dimension whose factor test is nonzero, this caller supplies:

| Item | Established contract |
|---|---|
| First argument | Signed 32-bit dimension–factor product plus 36, where both operations are representable |
| Second argument | 32-bit value 72 |
| Result consumed | One 32-bit working dimension |

Its operation, rounding, exceptional behavior, memory ownership effects, and shared-state effects are unknown.

The constants suggest a conversion or rounding purpose, but that is inferred intent only. A division operation, rounding convention, or error sentinel cannot be supplied as part of the established contract.

If the dependency does not return normally, this caller establishes no separate recovery behavior.

## Acceptance examples

These are conditional consequences of static inspection, not runtime test results. Unless stated otherwise, assume accessible storage, stable records, representable signed arithmetic, and dependency calls that return normally without modifying shared storage.

### Gate precedence

With the required context flag clear, the result is 35 even if the request discriminator would be nonzero. No direct request read or output write occurs.

With the flag set and a nonzero discriminator, the result is 7. Neither normalization call occurs.

### Empty table after normalization

With the gates satisfied and count zero or negative, the result is 23 without a table-entry read.

Any normalization call selected by a nonzero factor still occurs before this count check.

### Matching buckets

With zero factors and raw dimensions 100 and 200, the working buckets are:

- B(100) = 128.
- B(200) = 192.

An entry containing first dimension 110 and second dimension 210 matches because its buckets are also 128 and 192.

If indices 0 and 1 fail and this entry is at index 2, success returns zero and writes 2 when an output destination is supplied.

### Bucket boundaries and unsigned wraparound

B(31) = 0, while B(32) = 64. A one-unit difference can prevent a match.

B(32) and B(95) both equal 64, despite a difference of 63.

For signed value −1, its unsigned bit pattern plus 32 wraps to 31, so B(−1) = 0. It shares a bucket with zero. This wraparound follows from the comparison arithmetic, not from an assumed signed-overflow policy.

### One-sided relaxation and field access

For working dimensions 100 and 200, an entry containing 500 and 210 fails with relaxation zero but matches with any nonzero relaxation value.

The entry’s first-dimension field is still read in the relaxed case.

An entry whose second dimension is 224 fails regardless of relaxation: B(224) = 256, whereas B(200) = 192. Its first-dimension field is not read.

### Missing raw dimension

With zero factors, raw dimensions zero and 100 produce working dimensions 100 and 100. Raw dimensions 100 and zero produce the same result.

When both raw dimensions are zero, there is no substitution. If both factors are nonzero, two dependency calls occur, and their returned values remain independent unless other effects intervene.

### Conditional dependency result

Raw first dimension 10 with factor 2 supplies arguments 56 and 72 to F-870c4fcf7295.

If the dependency returns 90, the working first dimension initially becomes 90. With stable nonzero raw first and second dimensions, substitution leaves it unchanged.

### First match and null destination

If entries 1 and 3 both match and entry 0 fails, index 1 is selected. Entry 3 is not inspected.

A null destination does not alter selection or the zero success status.

### Changing count

Suppose index 0 is permitted by count 2 and fails. The index becomes 1. If the next count read yields 1, the function returns 23 without examining entry 1.

The count change affects traversal, but does not create index-increment overflow.

## Uncertainties

- **documented_only — Domain meaning:** Dimensions, factors, normalization, and missing-value interpretation are inferred roles; units and application semantics are unknown.
- **documented_only — Dependency behavior:** F-870c4fcf7295 has no supplied implementation contract beyond this caller’s arguments and consumed result.
- **documented_only — Normalization overflow:** Signed multiplication and addition overflow have no resolved portable behavior.
- **documented_only — Address arithmetic:** Signed entry-stride multiplication and address additions may exceed the representable range; no general wrapping policy is established.
- **documented_only — Invalid storage:** No defined error status is established for inaccessible, null, undersized, misaligned, or invalidated accessed storage.
- **documented_only — Shared-state mutation:** Dependency-induced and concurrent changes are unconstrained. Repeated reads do not form a consistent snapshot.
- **documented_only — Ownership and lifetime:** Surrounding allocation and ownership protocols are unavailable.
- **documented_only — Runtime behavior:** No runtime tests have been performed.

Index-increment overflow is not an uncertainty: the signed count check bounds every executed increment within the signed 32-bit range.

## Evidence

Established claims derive from static inspection of the supplied representation: access widths and offsets, gate order, conditional dependency calls, repeated factor reads, post-normalization substitution, unsigned bucket arithmetic, short-circuit comparisons, repeated count and base reads, ascending traversal, and the optional success-only write.

The index bound follows from the zero starting index, signed 32-bit count, strict inequality required to enter an iteration, and increment occurring only after a rejected permitted candidate. It does not require a stable count.

No dependency body, caller contract, complete record declaration, ownership protocol, or execution observation is supplied. Inferred purpose and unresolved behavior remain separate from established static evidence.
