<!-- Trusted provenance: F-e22ade46ea1c; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-e22ade46ea1c

## Purpose

F-e22ade46ea1c computes a bit-position result from a sequence of signed integers and two global multipliers. A scaled header determines the initial payload index. The function searches downward for a payload element whose scaled value is nonzero, then inspects that element’s scaled value for a set bit.

For stable inputs under a 32-bit integer model, with valid memory accesses and defined arithmetic, the result is the selected product’s one-based highest-set-bit position plus 32 positions for every preceding payload element.

Interpreting this as a scaled multiword bit-length operation is **inferred intent**. The established behavior does not define a complete numeric encoding, require normalized payload contents, or validate the sequence’s extent. Selection and bit inspection evaluate the selected product separately; they do not necessarily observe a single snapshot.

## Interface

### Argument and layout

The sole argument is a pointer to signed integer storage arranged as follows:

| Element index | Role |
|---|---|
| 0 | Header used to derive the search bound. |
| 1 onward | Payload elements considered by the descending search. |

The supplied representation uses signed integer elements, an unsigned bit counter, an initial bit position of 31, and a word-position stride of 32. These support a 32-bit interpretation, but no independent target ABI declaration is supplied. Exact integer widths are therefore **documented_only**.

Under the 32-bit model used for the conditional results in this contract:

- Each element occupies 32 bits.
- Payload element 1 immediately follows the header.
- Elements are four bytes apart on a target with eight-bit bytes.
- Indices identify integer elements, not byte offsets.
- Byte order does not affect the description of typed element accesses.

There is no length, capacity, end pointer, or allocation descriptor. The header-derived bound does not establish that the referenced storage exists.

### Global inputs

Two distinct global values participate:

| Global role | Use |
|---|---|
| Range multiplier | Multiplied by the header before forming the initial search index. |
| Payload multiplier | Multiplied by each candidate payload element and reevaluated during bit inspection. |

The visible arithmetic treats these as signed integer products. The globals’ declarations, initialization, permitted values, memory qualifications, and synchronization rules are not supplied.

### Return value

The return value is a signed integer.

For stable inputs and defined arithmetic:

- Zero indicates an empty search range or that every candidate product was zero.
- A positive result combines the selected element’s word offset with its product’s one-based highest-set-bit position.

There is no separate status output, explicit error return, or diagnostic channel. Zero is an ordinary result rather than evidence of rejected input.

### Preconditions for ordinary results

The ordinary result contract requires:

- A valid, suitably aligned pointer permitting the header read.
- Valid pointer indexing and readable storage for every payload element reached.
- Representable signed header and payload products.
- A representable magnitude of the header product.
- Defined intermediate arithmetic and a representable signed result.
- Stable relevant values, or an external execution model that defines any changes.

These conditions are not visibly checked.

Ownership is not a precondition established by the body: the storage may be caller-owned, borrowed, or owned by another party. Its lifetime and access permissions must nevertheless permit the reads performed.

## Behavior

### Bound formation

The function first multiplies the header by the range multiplier. If the signed result is negative, it negates that result. The resulting magnitude determines the initial candidate index.

Multiplication precedes sign normalization. It is not equivalent, for arithmetic-validity purposes, to taking the header’s magnitude first.

For stable values, define:

- \(N\): the magnitude of the signed header product.
- \(P_i\): the signed product of payload element \(i\) and the payload multiplier.
- \(U_i\): the unsigned interpretation of \(P_i\).

When \(N=0\), the function returns zero without evaluating a payload element.

The bound is computed once. Later changes to the header or range multiplier do not cause a new bound calculation.

### Descending candidate search

For a positive bound, candidate indices are considered in descending order, beginning at \(N\) and ending at 1.

Each candidate’s payload product is evaluated and compared with zero. A zero product advances the search to the next lower index. A nonzero product ends selection immediately.

For stable inputs, the selected index is therefore the largest \(i\) in the inclusive range \(1\) through \(N\) for which \(P_i\ne0\).

If every candidate product is zero, the function returns zero after considering element 1. The header at index 0 is not tested as a payload element.

Selection depends on the product, not merely on the stored element:

- A zero payload multiplier makes every defined candidate product zero.
- A zero payload element produces a zero product.
- Two nonzero operands produce a nonzero product when their signed multiplication is representable.
- Signed overflow has no established wraparound interpretation.

Once an element qualifies, lower-indexed payload elements are not inspected by this execution. Their contents do not contribute to the result.

### Bit inspection and reevaluation

The qualifying product is not retained as the fixed value used for bit inspection.

The function reevaluates the selected element’s product for the first bit test. Every subsequent unsuccessful bit test also reevaluates that product.

Each evaluated product is converted to unsigned form. Inspection starts at bit 31 and proceeds downward, stopping at the first tested bit that is set in that evaluation.

For a stable nonzero product under the 32-bit model:

- At most 32 bit tests are needed.
- The first successful test identifies the product’s highest set bit.
- A positive product contributes a within-word position from 1 through 31.
- A negative product converts to an unsigned value with bit 31 set and contributes a within-word position of 32.

The inspection is a bit-pattern operation, not a magnitude calculation for negative payload products.

With externally defined changes between evaluations, the successful bit test can depend on different product values at different stages. The function does not restart selection if the selected product changes or becomes zero.

### Result relationship

For stable values, let \(i\) be the selected index and \(b\) its unsigned product’s highest set-bit index, numbering the least significant bit as zero.

The ordinary mathematical result is:

\[
32(i-1)+b+1
\]

The offset accounts for preceding payload positions; it does not require reading those elements.

This formula is conditional on defined arithmetic and a representable signed return value. It does not replace the reevaluation behavior when inputs can change.

### Arithmetic boundaries

Under the 32-bit signed model, the representable range is −2,147,483,648 through 2,147,483,647.

No ordinary result is established for:

- Overflow in the header product.
- Negation of the minimum signed value, −2,147,483,648.
- Overflow in any candidate product.
- Overflow in any reevaluated product during bit inspection.

The word offset \(32(i-1)\) is calculated using signed arithmetic before it is combined with the unsigned bit counter. Later unsigned participation does not make an earlier signed overflow defined.

The largest selected index with a representable signed offset is 67,108,864. At that index, the offset is 2,147,483,616:

- A highest set-bit index of 30 produces 2,147,483,647, which fits the signed return type.
- A highest set-bit index of 31 produces the unsigned value 2,147,483,648 before conversion to the signed return type. The target-specific result of that conversion is **documented_only**.

A larger selected index overflows the signed offset multiplication under this model, even if the final expression might otherwise be interpreted as unsigned.

### Failure to find a set bit

The bit counter is unsigned. Its low eight bits supply the shift count.

During an ordinary successful scan, the counter ranges from 31 down to 0, so the masking does not change the shift count.

If every test through bit 0 fails, decrement wraps the unsigned counter to its maximum value. The next masked shift count is 255. This exceeds the valid shift range for a 32-bit operand.

Consequently, the mask does not provide safe recovery when a previously qualifying product becomes zero and remains zero throughout inspection. No defined return value or termination guarantee is established for that path.

## Side effects

### Direct accesses and ordering

The visible body:

1. Reads the header and range multiplier to establish the bound.
2. Evaluates candidate products in descending index order.
3. Reevaluates the selected payload product during bit inspection.
4. Returns an integer if execution reaches a return.

The relative read order of the operands within a multiplication is not established. The broader evaluation order is established: bound formation precedes candidate evaluation; selection precedes bit inspection; successful inspection precedes result formation.

These are logical evaluation requirements. Without declarations establishing volatile, atomic, or other special access properties, they do not specify an exact count of physical memory transactions.

### Writes, ownership, and lifetime

The body contains no visible:

- Writes through the argument.
- Assignments to either multiplier.
- Function calls.
- Allocation or release operations.
- Logging, callbacks, or explicit error-state updates.

**Storage ownership is unknown.** The argument’s presence does not establish that the caller owns the sequence.

The supported observation is narrower: the function contains no visible release of the storage, ownership transfer, or retention of the argument pointer. It does not visibly store the pointer in persistent state or pass it to another function. These observations do not establish a broader ownership protocol outside the supplied body.

No persistent state update is visible.

### Special memory and mutation

Read-only direct behavior does not establish that reads are externally effect-free. Device memory or other specially managed storage may attach effects to reads, but support for such storage is not established.

The representation also does not establish a legal concurrency mechanism. Unsynchronized mutation may invalidate execution rather than merely produce an unusual result.

## Dependencies

No called-function dependencies appear. No additional opaque function IDs are required.

| Dependency | Required contribution |
|---|---|
| Argument-addressed storage | Valid header access and valid access to each payload element reached. |
| Range multiplier | Determines the bound together with the header. |
| Payload multiplier | Determines candidate qualification and inspected bit patterns. |
| Integer execution model | Defines widths, conversions, arithmetic limits, and valid shifts. |
| External lifetime and synchronization rules | Determine whether accesses and any changes are valid. |

There is no unknown callee whose preservation of globals or treatment of memory must be assumed. The absence of calls does not resolve the ownership or lifetime of the argument-addressed storage.

## Acceptance examples

These are **conditional static acceptance examples**, not observed executions. Unless stated otherwise, assume 32-bit signed and unsigned integers, stable values, valid storage and pointer indexing, and defined arithmetic.

### Empty range

The range multiplier is 1 and the header is zero.

**Required result:** 0.

**Required access behavior:** no payload element is evaluated.

A zero range multiplier also produces an empty range for any valid header value.

### Negative header product

The header is −2, the range multiplier is 1, and the payload multiplier is 1. Payload element 2 is zero; payload element 1 is 8.

**Required behavior:** element 2 is evaluated before element 1, which qualifies.

**Required result:** 4.

### Selected element above the first payload position

The initial bound is 3 and the payload multiplier is 1. Payload element 3 is zero; element 2 is 4.

**Required behavior:** evaluate element 3, then element 2. Do not inspect element 1.

**Required result:** 35, consisting of a 32-position offset and a within-word position of 3.

### Payload scaling changes the bit position

The initial bound is 1, payload element 1 is 3, and the payload multiplier is 2.

**Required result:** 3, from product 6. The unscaled element’s bit length of 2 is not the result.

### Zero payload multiplier

The initial bound is a positive accessible \(N\), and the payload multiplier is zero.

**Required result:** 0.

**Required logical search:** evaluate candidate products from index \(N\) through 1 in descending order.

This is not an exact physical-load-count requirement and does not make invalid payload accesses acceptable.

### Negative selected product

The initial bound is 2, the payload multiplier is 1, and payload element 2 is −1.

**Required result:** 64.

Element 2 qualifies immediately. Its unsigned product has bit 31 set. Element 1 is not inspected.

### Range scaling determines the first candidate

The header is 2 and the range multiplier is −2.

**Required first candidate:** payload element 4.

Storage containing only the header and two payload elements is insufficient for that access.

### Signed return boundary

The initial bound is 67,108,864, the payload multiplier is 1, and the highest candidate is valid and qualifies immediately.

- If that element is 1,073,741,824, the required result is 2,147,483,647.
- If that element is −1, the unsigned result expression reaches 2,147,483,648. Its conversion to the signed return type remains **documented_only**.

### Change after qualification

Assume an external execution model validly permits the selected product to change after it qualifies.

**Required structural behavior:** retain the selected index and reevaluate its product during bit inspection; do not resume the descending candidate search.

If every bit test through bit 0 observes zero, the next shift count is invalid under the 32-bit model. No normal result is required.

### Borrowed storage

The argument designates readable storage owned by a third party and temporarily borrowed by the caller.

**Required ownership-related behavior:** the body performs no visible release, ownership transfer, or pointer retention.

The result otherwise follows the same arithmetic and access rules. No conclusion that ownership belongs to the caller is permitted.

## Uncertainties

The following remain **documented_only**:

- **Exact ABI:** independent declarations confirming integer widths and byte size are absent.
- **Ownership:** the storage’s owner, borrowing arrangement, and wider lifetime protocol are unknown.
- **Memory extent:** no allocation metadata or independent bounds validation is supplied.
- **Global properties:** initialization, allowed values, volatility, atomicity, and synchronization are unknown.
- **Overflow:** no portable result is established for overflowing signed products, minimum-value negation, or signed offset overflow.
- **Return conversion:** the target-specific result of converting an out-of-range unsigned value to the signed return type is unknown.
- **Mutation:** repeated evaluations are established, but a valid mechanism for changes between them is not.
- **Failed inspection:** no recovery or defined outcome is established after all valid bit positions test clear.
- **Physical access counts:** optimization constraints and exact memory-transaction counts are unknown.
- **Numeric encoding:** the body does not establish payload-sign restrictions, normalization requirements, or an allowed multiplier set.

These unknowns are not resolved by assuming caller ownership, wrapping signed arithmetic, benign concurrency, or a fixed selected-product snapshot.

## Evidence

**Established static evidence:** the supplied body forms a magnitude from a signed header product, searches descending payload indices for a nonzero signed product, reevaluates the selected product during unsigned bit inspection beginning at position 31, and combines the successful position with a signed 32-position word offset. No calls, writes, releases, ownership transfers, or pointer retention are visible.

**Inferred intent:** under stable, valid conditions, the result resembles a scaled multiword bit length.

**Unknown:** storage ownership, external lifetime guarantees, target-specific integer details, global access properties, and valid mutation mechanisms are not established by the body.

No runtime tests have been performed.
