<!-- Trusted provenance: F-e81ed1edd673; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-e81ed1edd673

## Purpose

F-e81ed1edd673 checks an encoded, length-delimited region in a supplied byte buffer. It validates an outer length, reads an entry count, and conditionally traverses that many subordinate regions. On acceptance, it returns the effective starting offset. Checked rejection paths return zero.

The function does not construct a decoded object, return a pointer, or visibly modify the supplied storage.

The apparent format is a 20-byte outer header followed by counted, length-delimited entries. This is an interpretation of the layout, not an additional validation rule:

- A zero entry count bypasses the final requirement that entries consume the outer region.
- Each entry must have at least 20 bytes remaining before its length is read, but its declared length may be less than 20.

The four-byte length interpretation below is conditional on wrapping machine arithmetic. The supplied representation includes a potentially overflowing signed multiplication in both outer-length and entry-length decoding. Consequently, the described decoding is not an unconditional portable original-language contract.

## Interface

### Inputs

**Descriptor reference.** A nullable reference to storage containing two consecutive 32-bit words:

| Byte offset | Representation and use |
|---|---|
| 0 | Buffer address representation; zero means absent |
| 4 | Extent factor used in the capacity expression |

Only these two descriptor words are visibly accessed.

**Requested offset.** An unsigned 32-bit offset. Zero selects an effective starting offset of 12. Every nonzero requested offset is used unchanged.

**Global input.** A scalar used as a capacity multiplier. Its value, declaration, units, and initialization lifecycle are not established.

### Output

The result is an unsigned 32-bit value:

- **Zero:** a checked rejection.
- **Nonzero:** acceptance, returning the effective starting offset.

There is no separate error code, diagnostic, or output object. Acceptance cannot return zero because requested offset zero is replaced with 12.

### Widths and arithmetic qualifications

The visible operations use:

- 8-bit stored bytes;
- a 16-bit unsigned entry count;
- 32-bit unsigned offsets, lengths, endpoints, and return values;
- signed integer intermediates in parts of the decoding expressions.

Unsigned 32-bit addition wraps modulo \(2^{32}\). Selected additions are followed by checks that reject wraparound; the preliminary offset-plus-12 calculation is not.

Two signed-arithmetic uncertainties must remain distinct:

1. **Capacity multiplication.** The product of the global multiplier and the descriptor’s extent factor is converted to an unsigned 32-bit capacity. The global declaration is unavailable. Portable behavior for an overflowing signed multiplication is not established by the subsequent conversion.
2. **Four-byte decoding.** The first stored byte is promoted to signed integer arithmetic and multiplied by the signed constant \(0x17000000\). With 32-bit signed integers, stored values from 0 through 5 produce representable products; values of 6 or greater overflow. Combining that result with unsigned terms afterward does not retroactively make the signed multiplication portable.

The modular decoding formula in this contract describes the wrapping-machine interpretation. Where the signed multiplication overflows, original-language behavior is **documented_only**, not resolved by that formula.

### Preconditions for non-faulting execution

The descriptor and every buffer address actually read must designate readable storage. The checks do not establish allocation validity, address-space validity, alignment requirements for descriptor access, or freedom from concurrent mutation.

The surrounding environment must supply any required global initialization and synchronization. Existing ownership of the descriptor and buffer is unknown; ownership by the caller is not a precondition established by this body.

## Behavior

### Descriptor screening and starting offset

The descriptor reference is checked first.

- A null descriptor returns zero without reading its fields.
- Otherwise, the first descriptor word is read.
- A zero buffer-address representation returns zero without reading the extent factor, capacity multiplier, or buffer bytes.

The nonzero buffer-address value is retained for later buffer accesses.

The effective starting offset is 12 for requested offset zero and otherwise equals the requested offset. Requested offset 1 therefore remains 1; there is no general minimum offset of 12.

### Preliminary capacity gate

The function evaluates the capacity expression using the global multiplier and descriptor extent factor. It requires:

\[
(\text{start}+12)\bmod 2^{32}<\text{capacity}.
\]

Equality rejects. On rejection, the function returns zero before reading the outer length.

There is no explicit wraparound rejection for this addition. A sufficiently large start can produce a small wrapped result and pass this gate. Such a pass does not establish that the subsequent buffer addresses are valid.

The order of the two operand loads within the capacity multiplication is not established.

### Encoded integers

Under wrapping machine arithmetic, define the decoded value of a stored byte as:

\[
D(b)=(23b)\bmod256.
\]

For four stored bytes, the resulting unsigned length is:

\[
L=D(b_0)2^{24}+D(b_1)2^{16}+D(b_2)2^8+D(b_3).
\]

The first stored byte contributes the most significant decoded byte.

This formula is conditional: the expression producing the most-significant term uses signed multiplication before combination with unsigned terms. Stored first-byte values of 6 or greater therefore encounter the signed-overflow uncertainty described in **Interface**. This applies independently to the outer length and every reached entry length. For first-byte values from 0 through 5, that particular multiplication does not overflow a 32-bit signed integer.

The count uses two stored bytes:

\[
N=D(b_{\mathrm{high}})2^8+D(b_{\mathrm{low}}).
\]

Its visible intermediate products fit within 32-bit signed arithmetic. Assignment to the 16-bit unsigned count retains the low 16 bits, yielding this two-byte interpretation.

Neither decoding operation writes decoded bytes back to storage. The relative load order of individual bytes within a field is not established.

### Outer-region checks

After the preliminary gate passes, four bytes beginning at the effective start are read as the outer length.

Subject to the decoding qualification, the function then requires:

1. The outer length is greater than 19.
2. The unsigned endpoint, computed as start plus outer length modulo \(2^{32}\), is strictly greater than start.
3. The endpoint is no greater than a newly evaluated capacity.

A length of 20 passes the first condition. An endpoint equal to capacity passes the third.

The strict endpoint-advancement check rejects wrapping outer additions: a positive length whose addition wraps cannot produce an endpoint greater than the original start.

The two capacity expressions are evaluated separately. The function does not establish a single cached capacity or a coherent snapshot across them. Stable inputs yield the same capacity, assuming defined multiplication; externally changing state need not.

The initial entry cursor is start plus 20. That arithmetic is evaluated before the second capacity comparison. After successful outer checks, the accepted outer length and nonwrapping endpoint ensure that this initial cursor does not wrap or exceed the endpoint.

### Count and header layout

Only after the outer checks pass does the function read the count:

| Offset relative to start | Meaning |
|---|---|
| 0–3 | Encoded outer length |
| 14 | High encoded count byte |
| 15 | Low encoded count byte |
| 20 | Initial entry cursor |

Other outer-header bytes are not read merely because they lie within the apparent header.

If the count is zero, the function returns the effective start immediately. It does not compare the initial entry cursor with the outer endpoint. An outer length greater than 20 can therefore be accepted without any entry traversal or complete-consumption check.

### Entry traversal

For each declared entry, the following validation stages occur in order:

1. Compute cursor plus 20 using unsigned 32-bit arithmetic.
2. Reject unless that result is strictly greater than the current cursor.
3. Reject if that result exceeds the outer endpoint.
4. Read four bytes at the current cursor and interpret the entry length, subject to the four-byte decoding qualification.
5. Compute the next cursor as current cursor plus entry length modulo \(2^{32}\).
6. Reject unless the next cursor is strictly greater than the current cursor.
7. Reject if the next cursor exceeds the outer endpoint.
8. Adopt the next cursor and decrement the remaining count by one.

Every rejection in this sequence returns zero.

The first two checks ensure that at least 20 bytes remain within the outer region before an entry length is read. They do not require the entry’s declared length to be at least 20.

A zero entry length fails the strict-advancement check. An entry addition that wraps also fails that check. Positive lengths from 1 through 19 can pass, provided the pre-read remaining-space gate and subsequent endpoint checks succeed.

Only the four length bytes are read for each entry. The function does not inspect the other bytes covered by the 20-byte remaining-space requirement. Short entries can cause successive nominal 20-byte header ranges to overlap.

### Final consumption

After a nonzero count has been exhausted, the final cursor must equal the outer endpoint.

- Equality returns the effective start.
- A final cursor below the endpoint returns zero.
- Overshoot has already been rejected during traversal.

This final equality requirement does not apply to the zero-count path.

## Side effects

The supplied body contains no visible writes to the descriptor, buffer, capacity multiplier, or other memory. It contains no visible function calls, allocation, deallocation, or ownership-management operation.

Pre-existing ownership is unknown. The absence of ownership-management operations does not establish that either supplied object belongs to the caller.

The established memory-read ordering is at validation-stage granularity:

1. Descriptor null screening.
2. Buffer-address field read and absent-buffer screening.
3. First capacity evaluation.
4. Preliminary capacity comparison.
5. Outer-length byte reads.
6. Outer-length threshold and endpoint-advancement checks.
7. Second capacity evaluation and endpoint-capacity comparison.
8. Count-byte reads.
9. For each reached entry, remaining-space checks before entry-length byte reads.

Computing offsets and cursors does not itself read buffer bytes.

No stronger ordering is established among individual byte loads in a compound decoding expression or between the operand loads of a capacity multiplication.

Malformed but readable data returns zero when it reaches a specified failing check. Inaccessible storage, invalid address formation, and unresolved signed-overflow cases are not established as checked zero-return errors.

## Dependencies

No external function dependencies are visible, so there are no additional opaque function IDs to enumerate.

F-e81ed1edd673 depends directly on:

- the descriptor’s first two 32-bit words;
- buffer bytes reached through the retained address;
- the global capacity multiplier;
- the environment’s integer, address, and memory-access semantics.

No callee behavior, global-preservation guarantee, allocation policy, or synchronization service is assumed.

## Acceptance examples

These are conditional static consequences, not observed executions. Unless stated otherwise, assume readable storage, stable inputs, valid address formation, and capacity arithmetic without unresolved overflow. Lengths and counts are decoded values. Four-byte fields must either avoid the signed-overflow case or be interpreted under wrapping machine arithmetic.

### Null descriptor and absent buffer

- A null descriptor returns zero without reading descriptor or buffer storage.
- A readable descriptor with a zero first word returns zero without evaluating capacity or reading buffer bytes.

### Offset selection

A requested offset of zero examines the region at offset 12 and returns 12 on acceptance.

A requested offset of 1 examines offset 1 and can return 1. It is not normalized to 12.

### Strict preliminary boundary

For start 12 and capacity 24, the preliminary comparison is \(24<24\), which fails. The function returns zero before reading the outer length, regardless of any readable data beyond that declared capacity.

### Minimum outer region

For start 12, capacity 64, outer length 20, and count zero, the endpoint is 32 and the result is 12.

Raw length bytes `00 00 00 0C` represent 20 because \(12\times23\bmod256=20\). Their first stored byte is zero, so the most-significant signed multiplication does not overflow. Raw count bytes `00 00` represent zero.

### Zero count with trailing content

For the same start and capacity, outer length 24 and count zero also return 12. The initial entry cursor is 32, while the endpoint is 36; zero count bypasses their equality check.

Raw length bytes `00 00 00 A8` represent 24 and avoid the most-significant-term overflow.

### Exactly consuming one entry

For start 12, capacity 64, outer length 40, count one, and entry length 20:

- The initial entry cursor is 32.
- The outer endpoint is 52.
- Twenty bytes remain when the entry is inspected.
- Advancement reaches 52.
- Final equality succeeds, returning 12.

Raw count bytes `00 A7` represent one.

### Positive count without entry space

For start 12, outer length 20, and count one, outer validation can pass, but the initial cursor already equals the endpoint. The remaining-space gate fails before any entry-length byte is read.

### Short entry accepted within a complete sequence

For start 12, capacity 64, outer length 44, count two, and consecutive entry lengths 4 and 20:

- The endpoint is 56.
- The first entry starts at 32, with 24 bytes remaining, and advances to 36.
- The second starts at 36, with exactly 20 bytes remaining, and advances to 56.
- Final equality succeeds, returning 12.

The length fields occupy offsets 32–35 and 36–39. This is an accepted short-entry case, not merely a short entry that survives one intermediate check.

### Short entry rejected only at final consumption

For start 12, outer length 40, count one, and entry length 4, the entry passes its remaining-space and advancement checks. The final cursor is 36 rather than 52, so the result is zero.

### Signed-overflow boundary in decoding

A reached four-byte field whose first stored byte is `05` does not overflow the most-significant signed multiplication. A first stored byte of `06` does.

Under wrapping machine arithmetic, the latter contributes decoded high byte 138 because \(6\times23=138\). Without that arithmetic interpretation, this contract assigns no portable decoded value or return result to that case. Subsequent bounds checks do not resolve the earlier overflow uncertainty.

## Uncertainties

The following remain **documented_only**:

- Semantic names, units, declaration, and lifecycle of the capacity inputs.
- Original-language behavior when capacity multiplication overflows signed arithmetic.
- Original-language behavior when either four-byte decoder’s first stored byte is at least 6 and its signed multiplication overflows.
- Exact machine address formation when buffer-address arithmetic exceeds the supported address space.
- The descriptor’s and buffer’s pre-existing ownership and lifetime arrangements.
- Concurrent-mutation permissions and synchronization.
- Individual load ordering within compound expressions.
- Whether the apparent headers correspond to an external protocol or file format.
- Whether zero-count trailing content and short entries are intended features or validation omissions.

No stricter format rule, ownership assignment, portable overflow result, or memory-safety guarantee is supplied to fill these gaps.

## Evidence

**Established static evidence:** The supplied body shows null and absent-buffer checks, zero-offset substitution, two capacity evaluations, conditional field decoding, outer-length and endpoint checks, count-controlled traversal, selected unsigned-wrap checks, the zero-count bypass, and final equality for nonzero counts. It shows no memory writes, calls, or ownership-management operations.

**Arithmetic evidence:** Both four-byte decoders form the most-significant term through signed multiplication by \(0x17000000\) before unsigned combination. Under 32-bit signed arithmetic, first-byte values of 6 or greater overflow that multiplication. The modular decoding formula therefore requires a wrapping-machine interpretation in those cases.

**Inferred intent:** A 20-byte outer header followed by counted length-delimited entries explains the field positions and cursor checks. It does not justify adding a minimum entry length or requiring consumption when count is zero.

**Verification status:** Static analysis only. No runtime tests have been performed. The acceptance examples are conditional deductions from the supplied body, not execution results.
