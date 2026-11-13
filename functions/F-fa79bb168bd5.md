<!-- Trusted provenance: F-fa79bb168bd5; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-fa79bb168bd5

## Purpose

F-fa79bb168bd5 handles a four-argument state-update request. Recognized requests either store a compact enumeration, store an unmodified 32-bit value, or store a Boolean in an indexed region of a caller-supplied object. Requests not completed locally are forwarded to F-375c8199e7c2.

Local recognition depends on the request category, selector, value, and several externally stored 32-bit values. Those external values are not supplied with the evidence. Consequently, some apparently familiar selector families cannot be treated as unconditionally recognized.

The established contract is the dispatch and storage behavior below. The intended application-level meaning of the stored fields is not established.

## Interface

The function has four inputs and no exposed return value:

| Input | Representation | Role |
|---|---|---|
| Object address | 32-bit address represented as an integer | Base of the object used for local state access; also forwarded unchanged |
| Category | 32-bit integer | Selects local dispatch or forwarding |
| Selector | 32-bit integer | Selects a field or selector family; may be normalized before forwarding |
| Value | 32-bit integer | Stored directly, translated to a byte, converted to Boolean, or forwarded |

Comparisons shown as ordering comparisons use the signed integer interpretation present in the supplied representation. Equality tests compare the corresponding 32-bit values.

For this contract, the externally stored values are assigned descriptive, document-local names:

| Name | Use |
|---|---|
| K1 | First selector boundary and two enumeration values |
| K2 | Selector for one direct 32-bit store |
| K3 | An enumeration value used by several fields |
| K4 | An enumeration value and the basis of another enumeration value |
| K5 | Two enumeration values used by later selector families |
| K6 | Second selector boundary |
| K7 | Forwarding classification for category `0x8861` |

These names do not assert values, immutability, or relationships between the external locations.

For local stores, define the **selected-region address** as:

**object address + `0x944` + signed index × `0x24`**

The index is read as a 32-bit integer from object offset `0x24c`. Field offsets below are relative to the selected-region address.

The `0x24` multiplier establishes an indexing stride, not a complete record layout. Recognized field offsets extend through `0x42`, beyond that stride. An implementation must not infer independent, nonoverlapping `0x24`-byte records from the multiplier alone.

### Preconditions

For a locally accepted request:

- The object’s index field must be readable.
- The computed destination must be writable for the store width.
- Address calculations must resolve to valid storage.
- Any alignment requirements of the actual target must be satisfied for 32-bit stores.

No index bounds check, pointer validation, destination-size check, or arithmetic-overflow check is visible.

For forwarded requests, additional preconditions may be imposed by F-375c8199e7c2; they are unknown.

## Behavior

### Dispatch and completion

Category `0x2300` enters the main selector dispatcher. A locally accepted request performs exactly one visible destination store and returns without calling F-375c8199e7c2.

Category `0x8861` reads K7 before testing the selector:

- Selector `0x8862` stores one byte at offset `0x42`: zero when Value is zero, one otherwise. It then returns locally.
- Any other selector is forwarded with K7 as the forwarding classification.

All other categories are forwarded with classification `0x500`.

A category-`0x2300` request that does not complete a local store is also forwarded with classification `0x500`. Depending on which selector family it entered, its forwarded selector may differ from its input selector.

### Main selector routing

For category `0x2300`, let the first boundary be K1 + `0x11`. The following routing rules preserve the precedence of the supplied evidence:

| Selector condition | Destination behavior |
|---|---|
| Equal to the first boundary | Four-value family with selector origin `0x8588`, field origin `0x2b` |
| Less than the first boundary, and equal to K2 | Direct 32-bit store at offset `0x34` |
| Less than the first boundary, greater than K2, and one of `0x8580`, `0x8581`, `0x8582` | Four-value family with selector origin `0x8580`, field origin `0x28` |
| Less than the first boundary, and less than K2 | Lower-selector rules described below |
| Greater than the first boundary, and equal to K6 | Four-value family with selector origin `0x8590`, field origin `0x2e` |
| Greater than the first boundary, less than K6, and one of `0x8589`, `0x858a` | Four-value family with selector origin `0x8588`, field origin `0x2b` |
| Greater than the first boundary, less than K6, and one of `0x8590`, `0x8591` | Four-value family with selector origin `0x8590`, field origin `0x2e` |
| Greater than both the first boundary and K6, and one of `0x8598`, `0x8599`, `0x859a` | Two-value family with selector origin `0x8598`, field origin `0x31` |

Selectors not covered by a reachable rule are forwarded unchanged.

The boundary-equality rules accept the actual boundary selector. There is no additional requirement that K1 + `0x11` equal `0x8588`, or that K6 equal `0x8592`. Therefore, destination offsets for those rules remain arithmetic expressions unless the external values are known.

The tabulated selector sets describe the ordinary nonoverflowing interpretation of the arithmetic. The evidence’s subtraction-based checks for `0x858a` and `0x8591` are subject to the arithmetic uncertainty described below.

### Lower-selector rules

These rules are reachable only when Selector is less than both K1 + `0x11` and K2.

| Selector | Behavior |
|---|---|
| `0x0d1c` | Store Value unchanged as 32 bits at offset `0x38` |
| `0x2200` | Translate Value to one byte at offset `0x25` |
| `0x8571` | Translate Value to one byte at offset `0x26` |
| `0x8572` | Translate Value to one byte at offset `0x27` |
| Any other value | Forward unchanged |

For selectors other than `0x0d1c` and `0x2200`, the supplied representation evaluates K4 + `0x8d` before testing for `0x8571` and `0x8572`. Thus an unrecognized lower selector can still cause a read of K4.

### Enumeration at offset `0x25`

For selector `0x2200`, the following rules are tested in the listed order:

| Value condition | Stored byte |
|---|---:|
| Equal to `0x2100` | 4 |
| Less than `0x2101`, and equal to `0x0104` | 3 |
| Less than `0x2101`, and equal to `0x0be2` | 2 |
| Less than `0x2101`, and equal to K3 | 1 |
| At least `0x2101`, and equal to `0x2101` | 0 |
| At least `0x2101`, and equal to `0x8570` | 5 |

Otherwise, the request is forwarded unchanged.

The ordering matters if K3 duplicates an earlier constant. The ordering condition also matters: equality to K3 alone does not establish acceptance.

### Enumeration at offset `0x26`

For selector `0x8571`, define the value boundary as K4 + `0x8d`.

Equality to that boundary is tested first and stores byte 3. Otherwise:

- Above the boundary, `0x8575` stores 4, `0x86ae` stores 6, and `0x86af` stores 7, in that order.
- Below the boundary, `0x0104` stores 2, K3 stores 0, `0x2100` stores 1, and K4 stores 5, in that order.

A value not accepted in its applicable range is forwarded unchanged.

In particular, `0x86ae` and `0x86af` are not unconditional accepted values: they must be above the boundary and must not have matched boundary equality.

### Enumeration at offset `0x27`

For selector `0x8572`, equality to K4 is tested first and stores byte 5. Otherwise:

- Above K4, equality to K4 + `0x8d` stores 3; otherwise `0x8575` stores 4.
- Below K4, `0x0104` stores 2, K3 stores 0, and `0x2100` stores 1, in that order.

Other values are forwarded unchanged. This selector does not have local mappings to bytes 6 or 7.

### Four-value families with origins `0x8580` and `0x8588`

For either family, the destination is:

**field origin + Selector − selector origin**

The selector and field origins are those in the main routing table. The function stores one byte using this ordered translation:

| Value | Stored byte |
|---|---:|
| `0x1702` | 0 |
| K1 − 1 | 1 |
| K1 | 2 |
| `0x8578` | 3 |

The first matching row wins.

If none matches, there is no local destination store. The request is forwarded with its selector replaced by:

**Selector − selector origin**

Thus the selector passed to the dependency can be a small family-relative number rather than the original external selector. The original Value remains unchanged.

### Four-value family with origin `0x8590`

The destination byte offset is:

**`0x2e` + Selector − `0x8590`**

The ordered translation is:

| Value | Stored byte |
|---|---:|
| `0x0300` | 0 |
| `0x0301` | 1 |
| K5 − 1 | 2 |
| K5 | 3 |

The first matching row wins. An unmatched value causes forwarding with selector replaced by Selector − `0x8590`.

### Two-value family with origin `0x8598`

The destination byte offset is:

**`0x31` + Selector − `0x8598`**

Equality to K5 − 1 stores byte 2. Otherwise, equality to K5 stores byte 3.

An unmatched value causes forwarding with selector replaced by Selector − `0x8598`. Values `0x0300` and `0x0301` have no independent local mapping in this family.

### Forwarding contract

Every forwarding path makes one call to F-375c8199e7c2 with five arguments, in this order:

1. Original object address.
2. Classification: K7 for category `0x8861`, otherwise `0x500`.
3. Original selector, except for the family-relative normalization described above.
4. Original Value.
5. Original Value again.

The incoming Category is not forwarded as an argument.

No local destination store precedes forwarding. This says nothing about changes made by the dependency.

## Side effects

A locally accepted request reads the 32-bit index at object offset `0x24c`, computes its destination, and writes either one byte or four bytes. It does not visibly read the previous destination value.

The direct stores at offsets `0x34` and `0x38` preserve the supplied Value’s 32-bit representation. Byte enumeration stores contain the explicit small values above, rather than truncations of Value. The Boolean store at offset `0x42` normalizes every nonzero Value, including negative values, to one.

No allocation, deallocation, ownership transfer, reference-count operation, or persistent pointer retention is visible in the local paths. The function borrows access to the supplied object; the evidence does not establish ownership of that object.

Relevant external values are consulted during dispatch before the successful local destination access. K7 is read even on the locally handled `0x8861`/`0x8862` path. On unsuccessful main-dispatch paths, the index need not be read at all.

The evidence contains repeated references to some external locations. It does not establish atomicity, volatile semantics, a coherent snapshot, or behavior under concurrent mutation. Similarly, it establishes no synchronization around the index or destination.

Observable effects and ownership behavior inside F-375c8199e7c2 remain unknown. It must not be assumed to preserve global state, preserve the object, or refrain from releasing memory.

## Dependencies

**F-375c8199e7c2** is the sole visible function dependency.

Its established role here is to receive requests not completed locally, using the five-argument arrangement above. Its return value, if any, is not exposed by F-fa79bb168bd5.

Its validation rules, error signaling, memory effects, ownership rules, and interpretation of the duplicated Value arguments are documented_only.

K1 through K7 are external data dependencies, not callable dependencies. Their values, initialization, mutability, and semantic names are unknown.

## Acceptance examples

These are conditional specification examples, not results of runtime execution. Unless otherwise stated, they assume valid object storage and nonoverflowing address arithmetic.

1. **Boolean normalization without main-dispatch constants.**  
   With Category `0x8861`, Selector `0x8862`, and Value −1, K7 is read and byte 1 is stored at selected-region offset `0x42`. No dependency call occurs. With Value zero, the stored byte is zero.

2. **Alternate-category forwarding.**  
   With Category `0x8861` and any Selector other than `0x8862`, F-375c8199e7c2 receives the original object, K7, the original Selector, and two copies of Value. No local destination store occurs.

3. **Unrecognized category.**  
   With Category `0x2301`, the request is forwarded with classification `0x500` and its original Selector, even if that Selector and Value would have been accepted under category `0x2300`.

4. **Unmodified 32-bit storage.**  
   Suppose Selector equals K2 and is less than K1 + `0x11`. Category `0x2300` then stores all 32 bits of Value at offset `0x34`, with no enumeration validation.

5. **Selector-family success.**  
   Suppose `0x8581` is less than K1 + `0x11` and greater than K2. With Category `0x2300`, Selector `0x8581`, and Value `0x1702`, byte zero is stored at offset `0x29`.

6. **Selector-family fallback normalization.**  
   Under the same routing assumptions, choose a Value unequal to `0x1702`, K1 − 1, K1, and `0x8578`. No destination byte is written. The dependency receives classification `0x500` and selector 1, followed by two copies of that Value.

7. **Range-gated enumeration.**  
   Suppose selector `0x8571` reaches the lower-selector rules. Value `0x86ae` stores byte 6 at offset `0x26` only when it is greater than K4 + `0x8d`. If it equals that boundary, byte 3 wins instead.

8. **Boundary routing overrides familiar selector interpretation.**  
   If K1 + `0x11` equals `0x2200`, Category `0x2300` with Selector `0x2200` enters the family whose origin is `0x8588`. It does not use the offset-`0x25` enumeration.

9. **No implied index rejection.**  
   A negative index is not explicitly rejected. Its calculated destination may precede the nominal region base. Successful access requires valid storage there; the evidence provides no defined error result for an invalid destination.

## Uncertainties

- **documented_only — External constants:** No numerical values for K1 through K7 are available. Conventional enum names or expected API values cannot replace them.
- **documented_only — Arithmetic model:** Inputs and index are represented as 32-bit integers, and local addresses use integer arithmetic. Exact behavior for signed overflow, address wraparound, and overflowing boundary calculations is not established. The formulas above are definitive where arithmetic is representable.
- **documented_only — Platform representation:** Byte order, pointer-conversion rules, and alignment fault behavior are not specified. Four-byte stores preserve the input representation according to the target.
- **documented_only — Failure behavior:** No local validation or explicit error-reporting mechanism appears. Invalid memory accesses, dependency failures, and abnormal termination are outside the established contract.
- **documented_only — Concurrency and aliasing:** No guarantees cover concurrent changes or aliasing between destinations, the index field, and external data.
- **Inferred intent:** The structure resembles an indexed configuration setter with compact internal encodings and a fallback handler. This interpretation does not establish domain semantics.

## Evidence

Established static evidence consists of the supplied representation of F-fa79bb168bd5 and its mapped call to F-375c8199e7c2. It supports the dispatch precedence, widths, address formulas, ordered translations, selector normalization, and dependency arguments specified above.

No external constant definitions, dependency implementation, caller contracts, object-layout declaration, or target ABI documentation were supplied. No runtime tests have been performed.
