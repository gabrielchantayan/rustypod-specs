<!-- Trusted provenance: F-5028f4afa8de; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-5028f4afa8de

## Purpose

F-5028f4afa8de evaluates a bounded predicate over the least significant bit of a selected table word.

On normal completion, the result is one exactly when the supplied unsigned index is strictly below the descriptor’s unsigned limit and the selected word’s least significant bit is clear. Otherwise, the result is zero.

This is a bit test, not a test that the entire selected word is zero. Nonzero words with a clear least significant bit satisfy the predicate.

The application-level meaning of the bit is unknown. Describing it as an allocation, availability, validity, or occupancy indicator would add intent not established by the supplied representation.

## Interface

### Inputs and result

| Item | Contract |
|---|---|
| Descriptor address | Identifies storage containing a table-base field and a limit field. |
| Index | An unsigned integer used for both the range decision and entry selection. |
| Result | A four-byte value containing zero or one on normal completion. |

The result has no separate error encoding. Zero can mean either that the index was rejected by the range decision or that an accepted index selected a word whose least significant bit was set. The caller cannot distinguish these cases from the result alone.

### Recovered storage model

The following layout uses the recovered 32-bit integer model. The supplied representation supports this interpretation, but does not independently establish the complete original ABI.

| Descriptor byte offset | Width | Role |
|---|---:|---|
| 0 | 4 bytes | Integer representation of the table base |
| 4 | 4 bytes | Not read by this function |
| 8 | 4 bytes | Limit, interpreted as unsigned |

Table entries occupy four bytes each, with a four-byte stride. Selection is relative to the stored table base, not relative to the descriptor address.

The base field is obtained through an integer read and participates in unsigned address arithmetic. It is not evidence of an inline table or of a native pointer field whose width is independent of the integer width.

The selected entry is read as an unsigned word. Only its least significant bit contributes to the result. This refers to the numerical low-order bit of the word, not necessarily to a particular byte at the lowest address; byte order is not independently established.

### Preconditions for normal completion

The descriptor’s limit field must be readable for every invocation.

When the index is below the limit, the base field and the computed selected-entry address must also be readable with the required word-access semantics. The function supplies no validation of those addresses, their alignment, their lifetime, or the amount of table storage.

A valid limit field alone is therefore insufficient to establish a valid invocation on the accepted-index path. Conversely, rejection of the index does not require a usable table base or table entry.

These are conditions for the described normal result, not checks performed by the function. Environment-specific behavior for invalid memory remains unspecified.

## Behavior

### Range decision and conditional access

The descriptor’s limit is interpreted as unsigned, and the index must be strictly smaller than it.

Equality is rejected. A limit of zero rejects every index. Under the recovered 32-bit model, an index of 4,294,967,295 is rejected for every representable limit.

A limit whose stored representation would be negative under signed interpretation is not treated as a negative bound. In particular, the all-ones 32-bit representation becomes an unsigned limit of 4,294,967,295.

The range decision controls whether any table-related access is needed. If it rejects the index, the normal result is zero without a source-level read of the base field or a selected entry.

This conditional access is part of the behavioral contract. An implementation that unconditionally reads the base or table before rejecting an index would impose memory-access requirements absent from the supplied representation.

### Address selection and arithmetic width

An accepted index selects the word at a displacement of four times the index from the stored base.

Under the recovered 32-bit model, this calculation uses unsigned 32-bit arithmetic. The integer base participates through conversion to that unsigned arithmetic domain. A base representation that would be negative as a signed integer is consequently not, by itself, rejected.

Both the scaled displacement and the addition to the base are reduced modulo 4,294,967,296. This matters independently of the range decision:

- A sufficiently large accepted index can produce a wrapped displacement.
- Adding a smaller displacement to a sufficiently high base can also wrap.
- A limit larger than the index does not establish that either calculation avoids wrapping.
- The resulting integer still requires an environment-supported conversion to an address and a valid word read.

No checked array boundary, allocation-size check, overflow rejection, or address-validity check is established. Caller-maintained invariants may prevent problematic calculations, but no such invariants are supplied.

The modular calculations describe the recovered integer model. Whether every resulting integer can designate readable memory is a separate, unresolved property of the target environment.

### Selected-word decision

For an accepted index, the selected word determines the result:

| Selected word property | Normal result |
|---|---:|
| Least significant bit clear | 1 |
| Least significant bit set | 0 |

All higher bits are irrelevant to the returned value. Zero satisfies the predicate, but so does every other representable even word. Every odd word fails it.

There is no established interpretation of any other bit and no normalization or modification of the selected word.

### Ordering and consistency

At the source-expression level, the limit is needed before deciding whether to access the base and table. On the accepted-index path, the base is needed to determine the selected-entry address before that entry can be read.

The descriptor field at byte offset four is not consulted. Rejection avoids the base and entry reads; acceptance requires both.

These are evaluation dependencies and conditional-access properties, not a complete machine-level memory trace. The supplied representation does not establish volatile accesses, atomic operations, barriers, or restrictions on processor speculation.

The function also does not establish a coherent snapshot of the descriptor and table. Concurrent changes cannot be assigned a stronger consistency guarantee from this body alone. Whether such concurrent access is valid at all depends on memory-model and synchronization information not supplied here.

No disjointness requirement between the descriptor and table is explicit in the body. An entry may coincide with descriptor storage if the resulting accesses are otherwise valid. Such overlap does not introduce writes or change the predicate.

## Side effects

There are no explicit memory writes. The function does not explicitly update the descriptor, table, or any identified global state.

There are no calls, allocations, releases, reference-count operations, or ownership transfers in the supplied body. Caller-provided storage is read without an established transfer of ownership. Any lifetime requirements needed for those reads remain the caller’s responsibility.

The returned value is the only explicit output. No diagnostic, exception translation, error-status field, or recovery action is present.

A required invalid read is not specified to produce zero. Depending on the execution environment, it may prevent normal completion or have other consequences. Those consequences are unresolved rather than additional predicate outcomes.

The absence of explicit writes does not prove that reads have no external effects when storage resides in special memory. Device-memory semantics and other environment-specific read effects are not established.

## Dependencies

No callable dependencies appear in the supplied body. No dependency function IDs are therefore required.

Normal behavior depends on:

- Readable descriptor storage for the accesses actually required.
- Readable selected-entry storage when the index is accepted.
- The recovered integer widths and unsigned conversion rules.
- Conversion of the calculated integer into a usable address.
- The target’s alignment, word-access, and memory-consistency rules.

There are no unknown callees whose memory ownership, release behavior, or preservation of global state must be assumed.

## Acceptance examples

These are conditional specifications, not execution results. Unless a case states otherwise, assume stable ordinary memory, valid required reads, and the recovered 32-bit model.

| Conditions | Required normal outcome |
|---|---|
| Limit zero; index zero. | Return zero without reading the base or a table entry. |
| Limit three; index three. | Return zero. Equality does not pass the range decision. |
| Limit three; index four. | Return zero without selecting an entry. |
| Limit three; index two; selected word zero. | Return one. |
| Limit three; index two; selected word two. | Return one despite the word being nonzero. |
| Limit three; index two; selected word three. | Return zero because the least significant bit is set. |
| Limit one; index zero; selected word 2,147,483,648. | Return one. The high bit does not affect the predicate. |
| Limit one; index zero; selected word 4,294,967,295. | Return zero. |

Additional boundary and access cases:

- **Unsigned limit:** An all-ones limit accepts index zero. The result then depends on the word selected through the stored base; signed-looking negativity does not reject the index.

- **Largest index:** Index 4,294,967,295 is rejected even when the limit is also 4,294,967,295. No selected entry is read.

- **Wrapped displacement:** With index 1,073,741,824 and limit 1,073,741,825, the range decision accepts. Scaling the index by four produces zero modulo 4,294,967,296. If the resulting base address is valid and readable, the word there determines the result.

- **Wrapped base addition:** With a base representation of 4,294,967,292, index one, and limit two, the calculated integer address is zero under the recovered arithmetic model. This is not automatically an error return or a successful read. A predicate result is specified only if conversion and access at that address are valid.

- **Unusable base with rejected index:** If the limit field is readable and the index is rejected, an unusable base representation does not prevent the specified zero result through a source-level base or table read; neither is required.

- **Unreadable entry with accepted index:** If the range decision accepts but the selected entry cannot be read validly, no normal return value is established.

- **Unused descriptor field:** Changing only the field at byte offset four cannot affect the computation through a descriptor-field read. If that same storage is also the selected table entry because of overlap, changing it can affect the bit test.

## Uncertainties

The following remain **documented_only**:

- The application meaning of the tested bit.
- Whether the limit represents capacity, logical length, or another boundary.
- The independently verified original ABI, integer definitions, pointer representation, alignment requirements, and byte order.
- The target-specific rules for converting the calculated integer to an address.
- Whether callers guarantee sufficient storage or prevent arithmetic wrapping.
- Whether descriptor and table storage overlap in actual use.
- The consequences of invalid, unaligned, or special-memory accesses.
- The legality and effects of concurrent mutation.
- Any synchronization or lifetime guarantees supplied by surrounding code.

These unknowns do not change the established conditional predicate. They limit which executions can be assigned a normal result and which recovered-model arithmetic examples apply to the original environment.

## Evidence

**Established static evidence:** The supplied representation shows a strict unsigned limit comparison that conditionally permits an indirect word read. Entry selection uses a four-byte scale. The selected word’s least significant bit determines whether an accepted index produces one or zero. The body contains no explicit writes or calls.

**Recovered-model interpretation:** Four-byte descriptor fields, the stated offsets, the 32-bit index boundaries, and modular address examples use the recovered 32-bit integer interpretation. They are not independent confirmation of the original target ABI.

**Inferred intent:** The function appears to expose a bounded flag predicate. No application-specific meaning for the flag or limit is established.

**Verification status:** Static review of the supplied representation only. No runtime tests, execution traces, or independent ABI verification have been performed.
