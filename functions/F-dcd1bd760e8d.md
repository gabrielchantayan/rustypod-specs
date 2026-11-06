<!-- Trusted provenance: F-dcd1bd760e8d; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-dcd1bd760e8d

## Purpose

F-dcd1bd760e8d selects a result using a key derived from its second input and a table referenced by its first input. A matching table entry takes precedence over fallback processing. The matched-entry path measures the selected data, requests storage, and copies the measured extent plus one byte into that storage. When the table ends without a match, a separate conversion path processes the second input.

The apparent intent is to return an independently stored, terminated representation, either from a table or through fallback conversion. That interpretation is **inferred intent**. The established behavior is the lookup, dependency-call sequence, allocation request, copy, failure reporting, and return selection described below. The meanings of the key, table entries, allocator, and conversion services are not independently established.

This function is not a validated-input interface. It performs no visible object-pointer, table-pointer, table-bound, or arithmetic-overflow checks.

## Interface

### Inputs

The visible interface has two 32-bit inputs:

| Input | Established use |
|---|---|
| Context address | Supplies the base address from which a table pointer is read at byte offset `0x34`. |
| Subject value | Passed to F-a13acf58cba0 to obtain a lookup key. If no table entry matches, its zero/nonzero value determines whether fallback conversion is attempted. |

The subject value is not visibly dereferenced by this function. Whether it is an address, handle, identifier, or another encoded value is determined by its dependencies and remains unknown.

A zero subject does **not** bypass initial key derivation or table lookup.

### Context and table layout

The visible representation uses 32-bit words and a 32-bit address space.

At byte offset `0x34` from the context address is a 32-bit table pointer. The table consists of records with a 12-byte stride:

| Record offset | Width | Meaning in this function |
|---|---:|---|
| `0x00` | 32 bits | Key compared with the derived lookup key. |
| `0x04` | 32 bits | Selected-data address; zero marks the end of the table. |
| `0x08` | 32 bits | Not read by this function. |

The second word controls termination. A record with a zero second word ends the search regardless of its key. No separate entry count is consulted.

The supplied representation does not establish byte order. The layout above specifies byte offsets and word widths, not serialized endianness.

### Output

The visible return value is a 32-bit value:

- On a successful matched-entry path, it is the nonzero value returned by F-43da6b23bd44 after the copy operation.
- On a successful fallback path, it is the nonzero result returned by F-a9c2069bb1a5.
- On an unmatched zero subject, it is zero.
- On the visible allocation or fallback failures, it is zero after the associated diagnostic call and, where applicable, cleanup.

These return descriptions assume that all invoked dependencies return normally. Exceptions, nonlocal exits, termination, or faults inside dependencies are not specified.

### Preconditions for ordinary completion

The visible accesses require:

- A readable context field at offset `0x34`.
- A readable table sequence extending through a matching record or a terminating record.
- Valid selected data for F-1cb585febd84 and the subsequent copy, when a record matches.
- Arguments accepted by every dependency invoked on the chosen path.

The function does not enforce these preconditions. An invalid address, unterminated table, or unsupported dependency argument has no defined recovery behavior in the supplied evidence.

## Behavior

### 1. Derive the key before accessing the table

F-a13acf58cba0 is called with the original subject value, including when that value is zero. Its 32-bit return value becomes the lookup key.

Only after that call returns does the function read the table pointer from context offset `0x34`. Consequently, if key derivation changes that context field, the later read observes the field as it exists after the call. The function does not snapshot the table pointer before key derivation.

No specific key value is treated as a key-derivation failure. In particular, a zero key is still compared against nonterminal table records.

### 2. Search in record order

For each record, the function first reads its second word.

- If that word is zero, the search ends without a match and fallback processing begins. The terminal record’s key is not compared.
- Otherwise, the function reads the first word and compares it with the derived key.
- If the keys differ, the next record is examined at an address 12 bytes later.
- If the keys agree, searching stops at that record.

This establishes first-match precedence. Later matching records are not considered.

The third word is neither examined nor used to advance the search beyond its contribution to the 12-byte record size.

There is no visible table-bound check, iteration limit, cycle detection, or address-wrap check.

### 3. Handle a matching record

After a match, the function reads the matched record’s second word again and treats that value as the selected-data address.

Under stable memory, this is the same nonzero value that allowed the record to be considered. The second read is nevertheless significant: if memory changes between the reads and the later value is zero, the function returns zero without measurement, allocation, fallback conversion, or a diagnostic call.

For a nonzero selected-data address:

1. F-1cb585febd84 is called with that address.
2. Its 32-bit result is increased by one to form the requested extent.
3. F-43da6b23bd44 is called with that extent and two zero arguments.
4. If the allocation result is nonzero, an external copy dependency is called with the allocation result as destination, the captured selected-data address as source, and the measured result plus one as the extent.
5. After the copy returns, the allocation result is returned.

The source address is captured before measurement and allocation. It is not fetched again from the table after either dependency call. This does not establish that the source contents remain unchanged.

The external copy result, if any, does not determine this function’s return value.

If F-43da6b23bd44 returns zero, the function calls F-4fc89ec41a6d with fixed leading arguments `(7, 102, 65, 0, 0)` and additional machine-state-derived arguments whose meanings are not established. If the diagnostic returns normally, this function returns zero.

A matched-entry allocation failure does **not** trigger fallback conversion.

### 4. Handle an unmatched subject

Once a terminal record is encountered, the original subject controls fallback behavior.

**Zero subject:** return zero without invoking F-f8891134cb93, F-a9c2069bb1a5, F-2642535311f0, or the fallback-failure diagnostic. Initial key derivation and table traversal have already occurred.

**Nonzero subject:**

1. Call F-f8891134cb93 with the subject and a second argument of zero.
2. If its result is nonzero, call F-a9c2069bb1a5 with that result.
3. Treat either a zero result from F-f8891134cb93 or a zero result from F-a9c2069bb1a5 as failure.
4. On failure, call F-4fc89ec41a6d with fixed leading arguments `(34, 121, 65, 0, 0)` and additional machine-state-derived arguments whose meanings are unknown.
5. Call F-2642535311f0 with the result of F-f8891134cb93, including when that result is zero.
6. Return the successful conversion result, or zero on failure.

If the first fallback dependency returns zero, F-a9c2069bb1a5 is skipped. The failure diagnostic precedes F-2642535311f0.

On fallback success, F-2642535311f0 still runs before the converted value is returned. Its return value, if any, is ignored.

### Integer-width and extent constraints

The measurement result and the visible addition use 32-bit quantities. No wider intermediate or overflow guard is shown.

The same measured-result-plus-one expression supplies the allocation and copy extents. The evidence does not establish:

- A nonnegative measurement result.
- A maximum supported measurement.
- Whether either callee interprets the extent as signed or unsigned.
- Valid behavior when adding one exceeds the applicable 32-bit range.

Accordingly, overflow cases are **documented_only**, not accepted large-object behavior. The function must not be described as guaranteeing sufficient storage for every possible measurement result.

## Side effects

### Visible reads and writes

The function visibly:

- Reads the context’s table-pointer field after key derivation.
- Reads table records in increasing 12-byte steps.
- Reads a record’s second word before its key.
- Rereads the selected record’s second word after a match.
- Invokes a copy into the nonzero destination returned by F-43da6b23bd44.

It performs no direct visible write to the context or table. This does not imply that dependencies preserve either object or any global state.

### Allocation and lifetime

The matched path returns the storage value obtained from F-43da6b23bd44 and does not release it locally. Caller ownership is a plausible interface intent, but the correct release operation and allocator domain are unknown.

The fallback path passes its intermediate result to F-2642535311f0 before returning the conversion result. The call pattern suggests intermediate-resource cleanup, but actual release semantics are not established.

In particular, the supplied evidence does not prove that:

- F-a9c2069bb1a5 returns separately owned storage.
- Its result remains valid after F-2642535311f0.
- The intermediate and final values cannot alias.
- F-2642535311f0 accepts zero safely.
- Any dependency preserves global state.

Those properties require dependency contracts and must not be assumed here.

### Diagnostics and failure handling

Two distinct visible failures invoke F-4fc89ec41a6d:

- Matched-entry allocation failure.
- Fallback acquisition or conversion failure.

Their fixed leading argument tuples differ. No human-readable message, exception policy, error-state update, or diagnostic severity is established.

The function does not visibly catch dependency failures or recover from invalid memory accesses.

## Dependencies

Only opaque identifiers are used for mapped dependencies.

| Dependency | Established interaction |
|---|---|
| F-a13acf58cba0 | Receives the original subject before table access; returns the lookup key. |
| F-f8891134cb93 | Receives a nonzero unmatched subject and zero; returns a fallback intermediate value. |
| F-a9c2069bb1a5 | Receives a nonzero fallback intermediate; its result becomes the fallback return value if nonzero. |
| F-4fc89ec41a6d | Receives one of two failure-specific fixed argument prefixes plus additional unspecified values. |
| F-43da6b23bd44 | Receives the measurement plus one and two zeros; a nonzero result is used as the copy destination. |
| F-1cb585febd84 | Receives the selected-data address; its result determines the requested extent after adding one. |
| F-2642535311f0 | Receives the fallback intermediate after success or diagnosed failure, including a zero intermediate. |

An additional external copy dependency is present, but no opaque identifier was supplied for it. Its identity and independent contract are **documented_only**. The call site establishes the destination, source, and extent arguments; conventional copying semantics are inferred intent rather than an independently verified dependency guarantee.

## Acceptance examples

These are conditional static expectations, not runtime test results. Unless stated otherwise, they assume valid stable memory, supported extents, and normally returning dependencies.

### First matching entry wins

Given two nonterminal records with the derived key, the earlier record supplies the selected-data address. The later record is not consulted. Measurement, allocation, and copying use the earlier selection.

### Terminal key does not match

Given a record whose key equals the derived key but whose second word is zero, the record terminates the search. It does not select a zero-address entry. Fallback behavior follows.

### Zero subject may still produce a matched result

Given a zero subject, a derived key, and a matching nonterminal record, the matched-entry path runs. Zero subject alone does not force a zero return.

### Unmatched zero subject

Given a zero subject and no match before the terminal record, the function returns zero after key derivation and traversal. No fallback acquisition, conversion, cleanup, or fallback diagnostic is invoked.

### Successful matched allocation

Given a selected entry, a measurement of four, and a nonzero allocation result, the allocation and copy extents are five. After the copy returns, the allocation result is returned.

If the measurement is zero, the requested extent is one; there is no visible special case for empty data.

### Matched allocation failure

Given a matching entry and a zero allocation result, the allocation-failure diagnostic is invoked and the function returns zero if that diagnostic returns. Fallback acquisition is not attempted.

### Failed fallback acquisition

Given an unmatched nonzero subject and a zero result from F-f8891134cb93, conversion is skipped. The fallback-failure diagnostic runs, F-2642535311f0 receives zero, and the function returns zero.

### Failed fallback conversion

Given a nonzero fallback intermediate and a zero conversion result, the fallback-failure diagnostic runs before F-2642535311f0 receives that intermediate. The return value is zero.

### Successful fallback conversion

Given a nonzero intermediate and a nonzero conversion result, no fallback-failure diagnostic runs. F-2642535311f0 receives the intermediate before the conversion result is returned. This example establishes call and return selection, not the returned object’s lifetime.

## Uncertainties

The following remain **documented_only**:

- Semantic types of the subject, keys, selected data, and fallback values.
- Whether measurement counts bytes, characters, or another unit.
- Whether the extra copied unit is a terminator.
- Allocator ownership, release API, alignment, and failure semantics beyond the visible zero check.
- Copy behavior, including overlap handling and argument interpretation.
- Dependency mutation of globals, context, table, selected data, or returned storage.
- Validity of fallback output after the intermediate cleanup call.
- Thread safety and concurrent-mutation behavior.
- Overflow behavior and signedness at dependency boundaries.
- Diagnostic side effects and the meanings of additional diagnostic arguments.
- Behavior on invalid addresses, missing table terminators, or dependencies that do not return normally.

The interpretation of the selected data as a terminated string is plausible but not required by the established static behavior.

## Evidence

This contract is based solely on the supplied static function representation and dependency-ID mapping.

Established observations include the 32-bit field usage, context offset, table stride, sentinel-before-key ordering, selected-address reread, dependency-call ordering, zero checks, fixed diagnostic prefixes, allocation/copy extent expression, and return-value selection.

String semantics, allocation ownership, and cleanup intent are inferences rather than independently established facts.

No runtime tests, dependency inspections, or execution traces have been performed. The acceptance examples are conditional predictions from static evidence, not observed executions.
