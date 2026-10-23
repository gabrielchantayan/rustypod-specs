<!-- Trusted provenance: F-c47e858a3dad; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-c47e858a3dad

## Purpose

F-c47e858a3dad performs a bounded, ordered search over entries obtained through an indexed-access dependency. It examines one unsigned byte in each returned entry and returns the address-like value associated with the first matching entry. It returns zero when the search bound is nonpositive or when no examined entry matches.

The search compares the complete unsigned 32-bit search value against a zero-extended byte. It does not reduce the search value to its low eight bits. Consequently, only search values from 0 through 255 can match an entry.

This is a static behavioral contract. The available evidence establishes the search mechanics but does not establish what the entries represent, whether the dependency accesses a conventional container, or who owns the returned storage.

## Interface

### Inputs

The function receives two arguments:

| Input | Representation | Required interpretation |
|---|---|---|
| Object base | A 32-bit signed integer used as an address-like value | Identifies storage containing a signed search bound and an embedded dependency argument region. |
| Search value | Unsigned 32-bit integer | Compared for equality against each examined unsigned 8-bit entry field, without truncation. |

The address-like first argument is not independently validated. A meaningful normal execution requires its relevant storage to be readable and its derived addresses to be valid in the execution environment.

### Object layout used by this function

Offsets are byte offsets relative to the object base.

| Offset | Use |
|---|---|
| 0x54 | Address of the region passed as the first argument to F-e7b12f377f06. This function does not directly read that region. |
| 0x58 | Location of the signed 32-bit search bound, read once before searching. |

These offsets describe accesses and arguments, not a complete object layout. The region beginning at offset 0x54 could contain the field at offset 0x58, but its structure is not established.

### Entry layout used by this function

For each value returned by F-e7b12f377f06, the function reads an unsigned 8-bit field at byte offset 0x3A from the returned address-like value.

No other entry fields are directly inspected.

### Output

The return representation is a signed 32-bit integer.

- On a match, the function returns the exact address-like value supplied by the dependency for that matching entry.
- On ordinary search exhaustion, it returns zero.
- On an initially nonpositive search bound, it returns zero without calling the dependency.

There is no separate status result or output parameter. The interface does not independently distinguish an unsuccessful search from a successful match whose dependency-returned value is zero. Whether such a match can execute successfully depends on the validity of the byte access derived from that value.

## Behavior

### Initial bound acquisition

The function first reads the signed 32-bit value at object offset 0x58. That value becomes the fixed bound for the entire invocation.

The bound is not refreshed between dependency calls. If a dependency call changes the stored bound, later loop decisions still use the value read at entry.

The initial candidate index is zero. The bound check precedes every dependency call:

- If the captured bound is zero or negative, the function returns zero immediately after the initial bound read.
- If the captured bound is positive, the function starts with index zero.

A nonpositive bound therefore suppresses entry access, but it does not suppress the initial object-field read. An invalid object address is not made safe merely because its intended bound would be nonpositive.

### Ordered candidate examination

For each index below the captured positive bound, the function performs the following observable sequence:

1. Calls F-e7b12f377f06 with the address at object offset 0x54 and the current index.
2. Uses that call’s returned value as an address-like entry base.
3. Reads the unsigned byte at entry offset 0x3A.
4. Compares the byte’s numeric value with the full unsigned 32-bit search value.
5. Returns the dependency’s value immediately if the values are equal.
6. Otherwise advances to the next index.

Indices are visited in ascending order, beginning at zero. No index is intentionally skipped or revisited. Once a match is found, no later dependency call or entry-field read is performed.

If all permitted candidates fail to match, the function returns zero after the final mismatch. For a captured positive bound of \(N\), normal exhaustion entails exactly \(N\) dependency calls, with indices zero through \(N-1\), and exactly one direct entry-byte read following each call.

### Equality and integer widths

The entry field has a value in the range 0 through 255. Its comparison value is zero-extended; values from 128 through 255 are not treated as negative.

The search value remains unsigned and 32 bits wide. For example:

- A search value of 255 can match an entry byte of 255.
- A search value of 256 cannot match any entry byte.
- A search value of 257 does not match an entry byte of 1.

A search value above 255 does not cause an early return. For a positive bound, the function still performs the ordered dependency calls and byte reads until exhaustion, unless execution is disrupted by an invalid access or dependency behavior.

The bound and index use signed 32-bit comparisons. For any positive representable bound, the index reaches the bound without requiring signed overflow. Even with the maximum positive signed 32-bit bound, the final increment following a mismatch reaches that bound, and the next bound check terminates the search.

Address arithmetic is separate from this index property. Validity of the derived object and entry addresses is a precondition for normal execution; address-overflow behavior is not established.

### Changes during dependency calls

Each entry byte is read after its corresponding dependency call returns. Thus, a write made by that call to the byte subsequently addressed can affect the comparison.

The function does not cache entry bytes before calling the dependency, nor does it re-read the matching byte before returning. It also does not revalidate the returned entry or the original object.

The evidence does not establish that the dependency preserves object contents, global state, or memory validity. The fixed captured bound specifies how the caller proceeds, but does not guarantee that entries remain accessible throughout the search.

## Side effects

### Direct effects

The function directly performs:

- One signed 32-bit read at object offset 0x58.
- An ordered sequence of calls to F-e7b12f377f06.
- One unsigned byte read at returned-entry offset 0x3A after each normally returning dependency call.
- Return of either zero or a dependency-provided value.

No direct writes to object storage, entry storage, or global storage are visible in the supplied evidence.

The absence of direct writes does not establish overall purity. Dependency calls can have effects not described here.

### Ownership and lifetime

The function contains no visible allocation, release, retain, or ownership-transfer operation. A successful result is the dependency-provided value, not a copied entry.

The evidence does not establish:

- Whether entries are borrowed or owned.
- Whether callers must release a returned entry.
- Whether the dependency allocates or releases memory.
- How long a returned entry remains valid.
- Whether repeated invocations return stable addresses.

These matters remain documented_only. Callers must not derive an ownership rule from this function alone.

### Errors and abnormal execution

There is no explicit null check, bounds validation of a dependency result, exception handling, retry, or error-code translation.

In particular, a zero dependency result is not treated as “entry absent.” The function attempts the byte read at the address obtained by adding offset 0x3A to that result. On systems where that address is inaccessible, execution cannot complete normally. The exact failure mechanism is outside the supplied evidence.

Likewise, if a dependency call fails to return normally, this function has no visible recovery path. Zero denotes an ordinary completed search with no match; it is not an established universal error result.

## Dependencies

### F-e7b12f377f06

This is the sole visible callee.

For each examined index, it receives:

1. The address at object-base offset 0x54.
2. The current nonnegative index.

Its returned value is used both as the base for the byte read at offset 0x3A and, if that byte matches, as the function’s final return value.

The caller’s use requires an accessible byte at the derived entry-field address for each examined candidate. However, the dependency’s actual guarantees are unknown. The evidence does not establish its internal indexing rules, range checks, failure representation, ownership behavior, or side effects.

Interpreting it as an indexed entry accessor is inferred intent, not an independently established dependency contract.

## Acceptance examples

These examples are conditional on normal dependency returns and valid memory for all described reads. They specify behavior; they are not runtime test results.

### Nonpositive bound

If the captured signed bound is zero or negative, the function returns zero. F-e7b12f377f06 is not called, and no entry byte is read.

The signed interpretation matters: a stored 32-bit pattern interpreted as a negative bound does not request a large unsigned search.

### Match after an earlier mismatch

Suppose the captured bound is three. The dependency’s index-zero result has byte value 7, and its index-one result has byte value 42. With search value 42, the function calls indices zero and one, reads their respective bytes, and returns the index-one result. It never requests index two.

### Multiple matching entries

Suppose indices zero and two would both produce byte value 9. With search value 9 and a sufficient positive bound, the function returns the index-zero result. The later matching candidate is not examined.

“First” means first in ascending dependency-index order, not lowest address or any other ordering.

### Exhaustion without a match

For a captured bound of two, if both examined bytes differ from the search value, the dependency is called with indices zero and one only. After the second mismatch, the function returns zero.

### Full-width search value

For a captured bound of two and search value 257, even two entry bytes equal to 1 do not match. Both candidates are examined, and the result is zero.

### Unsigned byte interpretation

If the first examined byte is 0xFF and the search value is 255, the first dependency result is returned. The byte is compared as 255, not as a negative value.

### Bound mutation

Suppose the bound is captured as three, and the first dependency call changes the stored bound to zero. If each candidate remains readable and no byte matches, the function still requests indices zero, one, and two before returning zero.

Conversely, increasing the stored bound during a dependency call does not extend the captured search range.

## Uncertainties

The following are documented_only:

- The semantic meaning of the object, entries, and compared byte.
- The complete layouts and alignment requirements of the relevant storage.
- Endianness of the stored 32-bit bound.
- Address-space behavior when address-like arithmetic exceeds its valid range.
- The dependency’s guarantees concerning valid indices, returned addresses, side effects, and failure.
- Ownership, lifetime, and release obligations for returned values.
- Thread safety, synchronization, and behavior under concurrent mutation.
- Platform-specific consequences of invalid memory access.

The supplied representation uses 32-bit integer values for address-like operations. That establishes the widths used in this contract, but does not by itself justify a complete platform ABI or a portable language-level pointer model.

## Evidence

Established static evidence consists of a single function body showing:

- A one-time signed 32-bit bound read at object offset 0x58.
- Ascending signed-index traversal starting at zero.
- Calls to F-e7b12f377f06 using object offset 0x54 and the current index.
- An unsigned byte read at returned-entry offset 0x3A.
- Equality against the untruncated unsigned 32-bit search value.
- Immediate return of the dependency result on a match.
- Zero return when the captured bound has been reached.

The interpretation as an entry search is inferred from those operations. Domain meaning and dependency guarantees are not established.

No runtime tests have been performed.
