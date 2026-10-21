<!-- Trusted provenance: F-faef215d09f0; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-faef215d09f0

## Purpose

F-faef215d09f0 performs a bounded, first-match lookup in a table referenced through shared state. It compares the supplied key against the key field of each record, in ascending record order. On the first match, it returns that record’s 16-bit result field. If none of the 205 records matches, it returns the 16-bit value zero.

The result does not distinguish a successful lookup whose stored value is zero from an unsuccessful lookup. Duplicate keys are resolved by record order, not by the associated result value.

This contract describes statically visible behavior. It does not assign application-specific meaning to the keys, table, or returned values.

## Interface

**Input**

- One integer key. The supplied representation supports a 32-bit integer interpretation; the original ABI and type declarations are not independently available.
- An implicit table base obtained from shared state. There is no explicit table pointer or length argument.

**Output**

- One 16-bit result.
- On a match, the result contains the bits read from the matching record’s result field.
- On exhaustion of the search, the result is zero.
- No separate status, match index, pointer, or error indicator is returned.

The result field is represented as an unspecified two-byte type. Signedness and application-level interpretation are therefore not established. A consumer must not infer that a set high bit indicates an error or that the result is necessarily an unsigned quantity.

**Table layout**

The lookup uses 205 logical records with a stride of 20 bytes:

| Record-relative offset | Access width | Role |
|---|---:|---|
| 0 | 4 bytes under the represented integer model | Key compared with the input |
| 10 | 2 bytes | Result read only when this record matches |
| Other offsets | No explicit access | Uninterpreted by this function |

The key and result are typed scalar accesses, not explicit byte-by-byte decoding. Byte order, alignment requirements, and the exact machine implementation of these accesses are not established by the supplied material.

**Preconditions for normal completion**

Every key field reached by the search must be readable. If a record matches, that record’s result field must also be readable. The shared table reference must permit those accesses under the target execution environment’s address and alignment rules.

These requirements depend on where the search terminates. An early match does not require this function to access later records. Conversely, an unsuccessful search requires all 205 key fields to be readable, but does not explicitly read any result field.

## Behavior

### Search extent and ordering

The search begins at record index 0. It visits consecutive indices in ascending order, stopping either at the first matching key or after examining index 204.

The bound is a record count, not a maximum key value. There is no key-range check, special handling for zero, or sentinel record that terminates the search early.

For record index \(i\), the key is located at byte displacement \(20i\) from the table base used for that access. The corresponding result is located at displacement \(20i + 10\).

Within the represented 32-bit arithmetic model, the index and displacement calculations remain small:

- The greatest visited index is 204.
- The greatest key displacement is 4,080 bytes.
- The last record’s result starts at displacement 4,090 bytes.
- After the last failed comparison, the index becomes 205 and the search terminates.

The index and stride multiplication do not approach 32-bit overflow. Validity of adding those displacements to the table base remains an environmental precondition.

### Comparison and first-match selection

Each visited record’s key is compared for exact integer equality with the input. No masking, normalization, range comparison, or alternate matching rule is visible.

If the comparison fails, the search proceeds to the next record without explicitly reading the failed record’s result field.

If the comparison succeeds, the function reads the two-byte result field associated with that record index and returns immediately. No later key or result field is explicitly read.

Consequently:

- An earlier matching record takes precedence over every later matching record.
- A matching result of zero is returned immediately; it does not cause the search to continue.
- An earlier duplicate with a zero result still hides a later duplicate with a nonzero result.
- Negative keys, under the represented signed integer interpretation, receive the same equality treatment as other keys.

### Observable access sequence

The source-level sequence is:

1. Obtain the table base needed for a key access and read the current record’s key.
2. Compare that key with the input.
3. On equality, obtain the table base needed for the result access, read that record index’s result field, and return.
4. Otherwise, advance the index and either repeat or return zero after the final failed comparison.

The table reference appears separately in the key-access and result-access expressions. The supplied material does not establish a stable snapshot of that reference across the lookup or even across the two accesses associated with a match.

If shared state changes during execution, no coherent-table guarantee follows from this function alone. In particular, a key comparison and its subsequent result access are not established as an atomic transaction. The exact consequences of concurrent modification depend on synchronization, memory rules, and compilation details not provided here.

These are source-level access statements. They do not establish physical bus transaction counts, compiler load-reuse behavior, or hardware speculation behavior.

### Exhaustion and failure behavior

If all 205 comparisons fail, the function returns zero after reading the key at index 204. It does not inspect index 205.

There is no visible validation of the table reference, accessible storage size, alignment, or initialization. There is also no visible recovery path for an invalid memory access. A fault or language-level consequence caused by invalid storage is not converted into the ordinary zero result.

## Side effects

No explicit memory writes are present. In particular, the function does not visibly:

- Modify the table or its shared reference.
- Cache a lookup result.
- Update a match counter or error flag.
- Allocate, release, or transfer ownership of storage.
- Invoke another function.

The function consumes access to existing storage without returning a reference to it. Nothing in the supplied material grants it ownership of that storage.

Its explicit memory effects are reads of the shared table reference and table fields. Such reads could have external consequences if the environment maps the storage to special-purpose memory; ordinary RAM versus device-backed storage is not established.

The function provides no visible locking or other synchronization. Callers requiring a consistent table must obtain that guarantee outside this function.

## Dependencies

No direct function-call dependencies are visible. No additional opaque function IDs can be assigned from the supplied dependency information.

The lookup nevertheless depends on external state and execution conventions:

- The shared table reference and its initialization.
- Storage containing the reached record fields.
- The target’s integer width, scalar representation, byte order, and alignment rules.
- Any synchronization needed to keep the table reference and contents consistent.
- The calling convention used to pass the integer key and convey the two-byte return value.

No assumptions about another function preserving globals, initializing the table, or owning its memory are justified by this body.

## Acceptance examples

These examples are conditional requirements derived from static evidence, not executed tests. Unless otherwise stated, assume readable storage, a stable table reference, stable record contents, and the represented integer widths.

| Condition | Required result and access boundary |
|---|---|
| Index 0 has the requested key and result `0x1234` | Return the 16-bit pattern `0x1234`; no later record is explicitly inspected. |
| Indices 0 through 6 fail; index 7 matches with result `0x002A` | Read keys in order through index 7, then return `0x002A` from index 7’s result field. |
| Indices 3 and 100 both match, with different results | Return index 3’s result; index 100 does not affect this invocation. |
| Index 3 matches with result zero; index 100 matches with a nonzero result | Return zero at index 3 without continuing. |
| No key among indices 0 through 204 matches | Return zero after the last comparison; no result-field read is explicitly required. |
| Only index 204 matches | Read all 205 keys in order, then read the result beginning at displacement 4,090 bytes. |
| A hypothetical record at index 205 matches, but the first 205 do not | Return zero; index 205 lies outside the search. |
| A matched result contains `0xFFFF` | Return those 16 bits; this contract assigns no signed or error meaning to them. |
| Index 0 matches and is readable, while later records are inaccessible | Normal completion requires no explicit access to those later records. |
| A reached key field is inaccessible | Normal completion is not guaranteed; zero is not a specified memory-error response. |

## Uncertainties

**Established by static evidence:** the 205-record bound, 20-byte stride, key and result offsets, first-match precedence, two-byte result, zero-on-exhaustion behavior, and absence of explicit writes or calls.

**Inferred intent:** the record structure and access pattern indicate a key-to-value lookup. The meanings of the key and value, and whether zero conventionally means “not found,” are not established. Zero is ambiguous at this interface regardless of any intended convention.

**documented_only:**

- Original ABI and confirmation of the represented 32-bit integer model.
- Table identity, lifetime, initializer, total allocation size, and actual contents.
- Semantic meaning of unused record bytes.
- Result signedness and caller interpretation.
- Byte order, required alignment, access atomicity, and exact machine-load behavior.
- Whether the shared reference or table can change concurrently, and what external synchronization exists.
- Behavior under invalid addresses, unsuitable alignment, address overflow, or other violated memory preconditions.

None of these unknowns is resolved by assuming a conventional caller, allocator, or initialization path.

## Evidence

The supplied static function representation shows an index initialized to zero, a key comparison at a 20-byte record stride, an immediate two-byte return from offset 10 on equality, and continued iteration while the advanced index is less than hexadecimal `CD`, equivalent to decimal 205. The exhaustion path returns zero.

The representation contains no explicit calls, memory writes, allocation, release, or error handling. These observations support the bounded lookup contract but do not establish surrounding system behavior.

No implementation access, runtime execution, or tests were performed.
