<!-- Trusted provenance: F-f874cbec901b; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-f874cbec901b

## Purpose

F-f874cbec901b coordinates an operation on a mutable context using an embedded input region, a caller-supplied payload word, and an opaque additional argument. After a prerequisite check, a lookup-like dependency determines whether the function attempts an existing-entry operation, performs fallback processing, or initializes a new entry.

Successful completion updates context metadata and writes a 16-byte result header. Failures after lookup invoke a cleanup dependency and clear that header. A prerequisite failure returns immediately without either action.

“Lookup,” “entry,” “creation,” and “capacity” are inferred roles, not established data-model semantics. The contract rests on observable accesses, dependency calls, branch conditions, and status transformations.

## Interface

### Arguments and return value

The function accepts four arguments:

1. **Context-reference address:** points to a 32-bit address-bearing word. The function reads this word after the prerequisite succeeds and retains the resulting context address. It does not subsequently reload the context reference.
2. **Record address:** identifies caller-provided storage containing a result header at offsets 0–15 and an embedded input region starting at offset 28.
3. **Payload-word address:** points to a 32-bit value read when preparing an explicit entry-operation or fallback call. The payload is not captured at function entry.
4. **Opaque argument:** a 32-bit value forwarded to the explicit entry-operation and fallback dependencies.

The return value is a 32-bit status. Zero identifies the common success return. Other returns are prerequisite results, lookup or subsequent dependency results, or locally selected statuses, subject to failure normalization.

The supplied representation uses 32-bit integer and address-bearing words, 16-bit halfwords, and 8-bit bytes. Exact ABI rules, native byte order, alignment requirements, and signed-overflow behavior are not independently established.

### Record layout

| Byte offset | Width | Direct access |
|---|---:|---|
| 0 | 32 bits | Receives the incremented context sequence on success; cleared on ordinary failure |
| 4 | 32 bits | Receives the current temporary location on success; cleared on ordinary failure |
| 8 | 16 bits | Cleared on success and ordinary failure |
| 10 | 16 bits | Cleared on success and ordinary failure |
| 12 | 32 bits | Cleared on success and ordinary failure |
| 16–27 | 12 bytes | No direct access |
| 28 onward | Unknown extent | Address passed as the embedded input region |

For each explicit entry-operation attempt, the function reads a length from record offset 28:

- If context flag `0x2` at offset `0x30` is clear, read one unsigned byte.
- If that flag is set, read one unsigned 16-bit halfword.

No byte-order conversion is visible. This selection is performed separately for each attempt, after preceding dependency calls. Fallback receives the embedded input address but no separately extracted length.

### Context fields

| Byte offset | Width | Direct use |
|---|---:|---|
| `0x02` | 16 bits | Read unsigned during fallback sizing; set to 1 after successful creation |
| `0x0C` | 32 bits | Set to the current location after successful creation |
| `0x10` | 32 bits | Incremented during common success processing |
| `0x14` | 32 bits | Set to the current location after successful creation |
| `0x18` | 32 bits | Set to the current location after successful creation |
| `0x20` | 32 bits | Read for capacity-like requests |
| `0x24` | 32 bits | Tested during creation and used in fallback arithmetic |
| `0x2C` | 32 bits | Read and rewritten with bit `0x1` set |
| `0x30` | 32 bits | Supplies the length-width flag |
| `0x34` | 32 bits | Incremented; its computed new value supplies record offset 0 |

These descriptions do not establish field names or the complete context layout.

### Temporary storage and preconditions

The function provides dependencies with a 128-byte work area, a five-word entry descriptor, a two-halfword selector area, and a 32-bit location slot. Only the first descriptor word is explicitly initialized, to zero. Other temporary contents are not initialized before lookup.

Dependencies must establish every temporary value subsequently consumed on the selected path. In particular, successful paths require a usable location value, and entry access requires a valid address in the first descriptor word.

All reached addresses must support the stated accesses and the requirements of invoked dependencies. No null, bounds, alignment, or aliasing checks are visible.

No direct allocation or deallocation operation is present. Dependency ownership, retention, and release behavior are unknown. Temporary addresses do not have an established lifetime beyond this invocation.

## Behavior

### Prerequisite and lookup

The function first initializes the first entry-descriptor word to zero and invokes F-ad862549a0f2 without explicit arguments.

A nonzero prerequisite result is returned immediately. On this path, the function does not directly read the context reference, access the record, or invoke cleanup. This does not constrain effects performed internally by the prerequisite dependency.

After a zero prerequisite result, the function reads and retains the context address. It then invokes F-5d3e17c1b8e7 with:

- The retained context address.
- The embedded input address at record offset 28.
- Addresses of the work area, location slot, entry descriptor, and selector area.

Immediately after lookup returns, the function reads a global 32-bit comparison status. This **first sample** is retained for ordinary failure normalization.

Routing then follows this precedence:

- Lookup result zero selects local failure status `0x21`.
- Lookup result `0x20` selects existing-entry processing.
- For any other lookup result, the function **reads the comparison global again** and compares the lookup result with this **second sample**. Equality selects creation; inequality selects ordinary failure with the lookup result.

The second read is therefore conditional: it does not occur for lookup results zero or `0x20`. No dependency call intervenes between the first and second reads, but global stability is not established. Creation selection uses the second sample; failure normalization uses the retained first sample. Later global values do not replace that retained normalization value.

### Creation

Creation proceeds through the following stages, stopping at the first failure:

1. Read context offset `0x24`. If zero, read context offset `0x20`, add one, and call F-308710cf24ed with the context and resulting request. A nonzero result selects ordinary failure. If `0x24` is nonzero, skip this call.
2. Call F-80c2f77a86cc with the context and location-slot address. A nonzero result selects ordinary failure.
3. Call F-b00cf3174ef8 with the context, current location value, and descriptor address. A nonzero result selects ordinary failure.
4. Treat the first descriptor word as an entry address. Write byte `0xFF` at entry offset 8, then byte `0x01` at entry offset 9.
5. Select the unsigned input length using the current context length-width flag.
6. Call F-dfe52f60212b with the context, entry address, selector zero, embedded input address, selected length, current payload word, and fourth argument.
7. Accept only result 1. Any other result is replaced by local failure status `0x23`.
8. After result 1, call F-f7c5b7585b2d with the context, descriptor address, zero, and `0x100`. A nonzero result selects ordinary failure.
9. After a zero post-attempt result, write context metadata in this order:
   - Set the halfword at `0x02` to 1.
   - Store the current location at `0x0C`.
   - Store the current location at `0x14`.
   - Store the current location at `0x18`.
   - Read the word at `0x2C` and rewrite it with bit `0x1` set.
10. Enter common success processing.

The location used after dependencies is the location then present in the temporary slot, not a value guaranteed unchanged since its first production. No local rollback of entry-byte writes or earlier effects is visible when a later stage fails.

### Existing-entry processing

The first selector halfword is interpreted as signed 16-bit.

A zero selector enters fallback directly. A nonzero selector, including a negative value, causes an explicit attempt:

- Select the unsigned input length using the current flag.
- Invoke F-dfe52f60212b with the same argument roles as creation, but with the signed first selector instead of zero.
- If the result is anything other than 1, enter fallback. That result is neither directly returned nor converted to `0x23`.
- If the result is 1, invoke F-f7c5b7585b2d with the context, descriptor address, zero, and `0x100`.

A zero post-attempt result enters common success. A nonzero post-attempt result enters ordinary failure; it does not trigger fallback.

### Fallback

Fallback forms a 32-bit difference from the unsigned halfword at context offset `0x02`, minus the word at `0x24`, plus one. The resulting value is tested as signed.

Only a positive difference causes sizing work:

1. Add the difference to context offset `0x20` to form a 32-bit candidate.
2. Call F-2d8bccefd701 with the context.
3. Compare its result with the candidate as unsigned values.
4. If the dependency result is smaller, increment the candidate once.
5. Call F-308710cf24ed with the context and candidate.

The adjustment is one increment, not replacement with a maximum or repeated growth. The candidate is computed before F-2d8bccefd701 and is not recomputed from context changes made by that dependency.

A nonzero F-308710cf24ed result enters ordinary failure. Otherwise, including when the signed difference was nonpositive, call F-cac7073e9138 with:

- Context address.
- Work-area address.
- Embedded input address.
- Current payload word.
- Fourth argument.
- Descriptor address.
- Current signed first selector.
- Fixed values 1 and 0.
- Location-slot address.

The selector and payload used for this call are read for this call; earlier dependency calls are not assumed to preserve them. A zero fallback result enters common success. A nonzero result enters ordinary failure.

### Common success

The direct operations occur in this order:

1. Read context offset `0x34`, add one, and store the computed value back.
2. Read context offset `0x10`, add one, and store the result back.
3. Read context offset `0x2C` and rewrite it with bit `0x1` set.
4. Store the computed new `0x34` value at record offset 0.
5. Store the current location value at record offset 4.
6. Clear the halfword at record offset 8.
7. Clear the halfword at record offset 10.
8. Clear the word at record offset 12.
9. Return zero.

The sequence value written to the record is the value computed in the first step, rather than a fresh context read after the intervening writes.

Creation has already set context bit `0x1`; common success performs a second read-modify-write of that field.

### Ordinary failure

All failures reaching this path perform these operations in order:

1. Invoke F-b0f922796692 with the context and descriptor address.
2. Clear the word at record offset 0.
3. Clear the word at record offset 4.
4. Clear the halfword at record offset 8.
5. Compare the pending failure status with the retained **first** global sample. If equal, replace the pending status with `0x20`.
6. Clear the halfword at record offset 10.
7. Clear the word at record offset 12.
8. Return the pending status.

Cleanup’s return value does not determine the function’s return. Global changes made during cleanup do not alter the retained normalization sample.

Normalization also applies to locally selected statuses. Thus `0x21` becomes `0x20` when the first sample was `0x21`, and `0x23` becomes `0x20` when the first sample was `0x23`.

## Side effects

Direct writes affect temporary storage, the specified context fields, the result header, and the two creation-entry bytes. Dependencies may perform additional memory writes or external actions; their scope is unknown.

Failure is not established to be transactional. Cleanup is called, but neither rollback nor resource release follows from that fact. Capacity changes, entry mutations, or other dependency effects may survive a failing return.

Header clearing occurs after cleanup. Record offsets 16–27 and the embedded input are not directly modified by this function. Dependencies receive the embedded input address, so the complete invocation does not guarantee their preservation.

Aliasing can affect subsequent accesses. For example, result-header writes that overlap context or temporary-referenced storage can invalidate ordinary final-state expectations. Independent valid buffers support the simple layout interpretation, but nonaliasing is not checked.

The stated stage ordering does not establish a left-to-right evaluation order for multiple memory reads used to prepare one dependency call.

## Dependencies

| Opaque ID | Contract-visible role |
|---|---|
| F-ad862549a0f2 | Prerequisite gate; nonzero returns immediately |
| F-5d3e17c1b8e7 | Lookup-like routing operation receiving temporary output areas |
| F-308710cf24ed | Capacity-like request; zero permits continuation |
| F-80c2f77a86cc | Creation-stage operation receiving the location-slot address |
| F-b00cf3174ef8 | Creation-stage operation receiving a location and descriptor address |
| F-dfe52f60212b | Explicit entry operation; exactly 1 is accepted |
| F-f7c5b7585b2d | Post-attempt operation with fixed arguments zero and `0x100`; zero succeeds |
| F-2d8bccefd701 | Supplies the unsigned comparison value for fallback sizing |
| F-cac7073e9138 | Fallback operation; zero succeeds |
| F-b0f922796692 | Ordinary-failure cleanup; return value ignored |

These roles do not establish preservation of globals, context fields, temporary contents, or caller memory. They also do not establish allocation, release, persistence, or ownership transfer.

## Acceptance examples

These are conditional static acceptance cases, not executed tests. They assume valid storage and no aliasing that changes the described outcome.

1. **Prerequisite rejection.** If F-ad862549a0f2 returns 7, the function returns 7 without directly reading the context reference or clearing the header.

2. **Zero lookup.** If lookup returns zero, no second comparison-global read occurs. Cleanup runs and the header is cleared. The return is `0x21`, except that a first sample of `0x21` normalizes it to `0x20`.

3. **Different global samples reject creation.** If lookup returns 7, the first sample is 7, and the second sample is 8, creation is not selected. Cleanup follows, and normalization changes the pending lookup status 7 to `0x20`.

4. **Different global samples permit creation.** If lookup returns 7, the first sample is 9, and the second sample is 7, creation is selected. The first sample does not prevent entry into creation. If a later creation dependency returns 9 as its failure status, normalization changes that status to `0x20`.

5. **Existing-entry precedence and success.** A lookup result of `0x20` selects existing-entry processing regardless of the first global sample, with no second global read. Given a nonzero selector, explicit-attempt result 1, and post-attempt result zero, common success follows. If context offset `0x34` equals 12 immediately before common success, its incremented value and record offset 0 are 13. Nonoverlapping storage alone does not establish that preceding dependencies preserved an earlier value of 12.

6. **Failed explicit attempt falls back.** On the existing-entry path, an explicit-attempt result zero enters fallback rather than returning zero or selecting `0x23`.

7. **Post-attempt failure does not fall back.** An explicit-attempt result 1 followed by post-attempt result 9 invokes cleanup. The return is 9 unless the first global sample was 9, in which case it is `0x20`.

8. **Fallback sizing.** With halfword `0x02` equal to 3, word `0x24` equal to 1, and word `0x20` equal to 10 when read, the difference is 3 and candidate is 13. A comparison result of 12 causes a request of 14; a result of 13 causes a request of 13.

9. **Length width.** A clear width flag and embedded first byte 5 produce length 5. A set flag and native unsigned halfword 261 produce length 261.

10. **Creation rejection after mutation.** A creation attempt returning anything other than 1 selects `0x23`, subject to first-sample normalization. The entry bytes have already been written; restoration is not established.

## Uncertainties

- **documented_only — Data semantics:** The underlying operation, full layouts, field meanings, and entry representation remain unknown.
- **documented_only — Global stability:** The two comparison-global reads need not agree. Visibility, synchronization, and external mutation mechanisms are not established.
- **documented_only — Dependency effects:** Callee contracts do not establish preservation, ownership, resource release, or rollback.
- **documented_only — Temporary outputs:** Lookup and later dependencies must supply consumed temporary values, but their guarantees are unavailable.
- **documented_only — Arithmetic boundaries:** Fallback subtraction and its added one use unsigned 32-bit arithmetic before signed interpretation. Candidate comparison and candidate increment are unsigned. Creation sizing, common increments, and the addition forming the fallback candidate are represented as signed additions; their overflow behavior is not independently established. Boundary behavior must not be generalized from ordinary mathematical examples.
- **documented_only — ABI and memory model:** Byte order, alignment tolerance, pointer validity, signed conversion details at boundaries, and supported aliasing arrangements remain unresolved.
- **Inferred intent:** The function resembles lookup followed by update or insertion with capacity management. Acceptance does not depend on that interpretation.

## Evidence

Established behavior derives from static inspection of the supplied representation: access widths and offsets, explicit initialization, dependency arguments, branch precedence, both comparison-global reads, direct write ordering, and status normalization.

The representation distinguishes the first global sample retained for failure normalization from the later conditional read used to select creation. It also reads the sequence field only upon entering common success; no supplied dependency contract establishes preservation of its earlier value.

No runtime tests have been performed. Acceptance examples are conditional consequences of static evidence, not observed execution results. No independent dependency implementations or external behavioral contracts were supplied.
