<!-- Trusted provenance: F-4c6b4c1ba53d; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-4c6b4c1ba53d

## Purpose

F-4c6b4c1ba53d conditionally prepares operations associated with a database object named `sqlite_stat1`. It obtains an opaque operation context, looks up the named object in a selected database, takes a creation or existing-object path, and makes a common sequence of calls using the resulting object identifier.

The supplied static representation establishes the calls and their ordering, but not the implementations of the dependencies. In particular, it does not establish that this function immediately creates, deletes, clears, or opens persistent database storage. The numeric operation selectors and SQL-like text suggest preparation of database work; that interpretation remains inferred intent rather than a guaranteed execution effect.

This function has no explicit return value or local error report. Whether dependency failures are recorded elsewhere, terminate execution, or permit subsequent calls is not established.

## Interface

The function accepts four arguments:

| Argument | Representation | Established use |
|---|---|---|
| State | Address of a structure accessed through 32-bit words | Supplies an initial connection-like value and, on the creation path, a later result word. Also passed to several dependencies. |
| Database selector | 32-bit integer | Selects a 24-byte record and is forwarded to operation-related dependencies. |
| Operation operand | Opaque 32-bit value | Forwarded unchanged to one common dependency call. |
| Filter operand | 32-bit integer or address-sized value in the represented environment | Zero selects the unfiltered existing-object path. Any nonzero bit pattern selects the filtered path and is forwarded to its formatting dependency. |

The represented environment uses 32-bit integer and address-bearing values. The interface does not establish portability to a wider-pointer environment.

The state has two directly observed fields:

- A 32-bit word at byte offset 0, read on entry. Its value is used as the connection-like base.
- A 32-bit word at byte offset 328, read only after the creation-path formatting call returns. This value becomes the identifier used by the common operation sequence.

The connection-like structure has a 32-bit word at byte offset 8. When the obtained operation context is nonzero, that word supplies the base address of an array of 24-byte records. The database selector selects a record, whose first 32-bit word is passed to lookup and formatting dependencies.

A successful, nonzero lookup result is treated as an address of an object with a 32-bit identifier at byte offset 20.

There is no demonstrated bounds checking, null checking, ownership transfer, allocation, or release in this function. Valid access to every directly read location is a caller or surrounding-system precondition for a normal execution of the corresponding path.

## Behavior

### Entry and context acquisition

The first observable data access is a read of the state’s word at byte offset 0. This happens before the call to F-9b3ea8470521.

F-9b3ea8470521 is then called without explicit arguments. Its result is treated as an opaque context value:

- If the result is zero, the function returns without making any further dependency calls.
- If the result is nonzero, processing continues with that context.

The zero-context path still requires the initial state read to be valid. It is not a path that permits an unreadable state pointer.

The connection-like value read before context acquisition is retained for subsequent address calculations and lookup. The function does not reread the state’s first word after F-9b3ea8470521 returns. Consequently, a dependency changing that state word would not change the connection-like value used by this invocation.

### Database-record selection and lookup

After obtaining a nonzero context, the function:

1. Reads the record-array base from byte offset 8 of the retained connection-like value.
2. Locates the database record using a stride of 24 bytes and the database selector.
3. Reads the first 32-bit word of that record.
4. Calls F-4e77314e8c36 with the retained connection-like value, the name `sqlite_stat1`, and the word just read.

The lookup result determines the next path:

- Zero selects the creation path.
- Any nonzero value selects the existing-object path.

The record address is retained across later calls. Where the creation or filtered path needs the record’s first word again, that word is reread at the retained address. The function does not recompute the record address from potentially changed state.

Thus, the lookup argument and a later formatting argument need not have the same value if a dependency modifies the selected record between those reads.

### Creation path

When lookup returns zero, F-66c61ffd0a02 is called with:

- The original state pointer.
- A supplied SQL-like format string whose visible prefix begins with `CREATE TABLE` and refers to `sqlite_stat1`.
- A fresh read of the selected record’s first 32-bit word.

The supplied representation does not expose the complete format string. It therefore does not establish the complete table definition, exact quoting rules, or final formatted statement.

After F-66c61ffd0a02 returns, the function reads the state word at byte offset 328. That word is used as the object identifier for the remaining calls. No local check verifies that the formatting dependency succeeded or that this word contains a valid identifier.

The creation marker is set to the 32-bit value 1.

This path does not call F-887ba2404a49 or F-a836e5479ebf. The filter operand does not affect creation-path branching or arguments.

### Existing-object path with a zero filter

When lookup returns nonzero and the filter operand is zero, the function:

1. Reads the 32-bit identifier at byte offset 20 of the lookup result.
2. Calls F-887ba2404a49 with the context, selector 94, the identifier, the database selector, and zero.
3. Calls F-a836e5479ebf with the state pointer, database selector, the same captured identifier, the literal value 1, and `sqlite_stat1`.

The identifier is captured before F-887ba2404a49. It is not reread before F-a836e5479ebf or before the common tail. Changes to the object’s identifier field during those dependencies would therefore not alter the identifier already captured by this invocation.

The creation marker remains zero.

Selector 94 may represent a whole-object operation, but its meaning cannot be established from this function alone. In particular, the call must not be described as proven immediate deletion or clearing of rows.

### Existing-object path with a nonzero filter

When lookup returns nonzero and the filter operand is nonzero, the function first calls F-66c61ffd0a02 with:

- The state pointer.
- A supplied SQL-like format string whose visible prefix begins with `DELETE FROM`, refers to `sqlite_stat1`, and includes `WHERE`.
- A fresh read of the selected record’s first 32-bit word.
- The original filter operand.

The full predicate, substitution rules, quoting behavior, and interpretation of the filter operand are not available.

Only after that dependency returns does the function read the 32-bit identifier at byte offset 20 of the lookup result. It then calls F-a836e5479ebf with the state pointer, database selector, captured identifier, literal 1, and `sqlite_stat1`.

This read timing is significant. If F-66c61ffd0a02 changes the identifier field while leaving the lookup-result address valid, the later calls use the changed value. If that dependency invalidates the object, this function contains no local lifetime check before dereferencing it.

This path does not call F-887ba2404a49. The creation marker remains zero.

### Common tail

Each path that obtains a nonzero context and reaches the common tail makes the following calls in order:

1. F-2ea344827b50 with context, selector 98, and operands 0, 3, 0.
2. F-2ea344827b50 with context, selector 9, the original operation operand, the captured object identifier, and the original database selector.
3. F-d2aa1dc1574b with context and the creation marker.

The creation marker is 1 only for the zero-lookup creation path. It is 0 for both existing-object paths.

No return value from these calls is inspected. Their operation-selector meanings and external effects remain dependency-defined.

### Arithmetic and validity boundaries

Record selection uses a 24-byte stride. No explicit validation rejects negative selectors, out-of-range selectors, multiplication overflow, or address overflow.

For an ordinary valid invocation, the selector must identify an accessible record and the address calculation must produce its intended address. The supplied representation does not establish language-level overflow behavior or justify promising modular indexing for extreme inputs.

The function also accepts any nonzero lookup result as an address without checking alignment, accessibility, type, or lifetime.

## Side effects

The function contains no explicit direct store to caller-visible memory. Its explicit memory accesses are reads.

That does not make it side-effect free:

- Dependencies receive the state pointer, retained connection-like value, context, identifiers, or record-derived values.
- Dependencies may modify state, global data, operation streams, or database-related resources.
- Later reads can observe dependency effects, notably the creation-path state word, repeated record-word reads, and the filtered-path identifier read.

No dependency is established to preserve global state or referenced memory. Observable ordering therefore includes both call order and the position of reads relative to those calls.

The function itself shows no allocation or release. Ownership of the context, lookup object, record array, formatting inputs, and any dependency-created resources is unspecified. Passing a value to a dependency does not establish transfer of ownership.

There is no local rollback, retry, cleanup-on-error branch, or explicit error propagation. The stated sequences describe execution when dependencies return normally and required reads remain valid.

## Dependencies

| Opaque ID | Role established at this call site | Unknowns |
|---|---|---|
| F-9b3ea8470521 | Produces the context whose zero/nonzero value gates further processing. | Acquisition semantics, errors, ownership, global effects. |
| F-4e77314e8c36 | Receives connection-like value, `sqlite_stat1`, and a selected-record word; returns zero or an address used as an object. | Lookup rules, object lifetime, mutations, error encoding. |
| F-66c61ffd0a02 | Receives state and SQL-like formatting inputs on creation or filtered existing-object paths. | Complete text, execution versus generation, success reporting, memory effects. |
| F-887ba2404a49 | Receives selector 94 and operands on the unfiltered existing-object path. | Operation meaning, immediate versus deferred effects, return semantics. |
| F-a836e5479ebf | Receives state, database selector, identifier, literal 1, and `sqlite_stat1` on both existing-object paths. | Authorization, locking, bookkeeping, or other intended role. |
| F-2ea344827b50 | Receives selectors 98 and 9 in that order on every completed nonzero-context path. | Selector semantics, operand interpretation beyond observed forwarding, failures. |
| F-d2aa1dc1574b | Receives context and the creation marker last. | Meaning of the marker, finalization effects, ownership consequences. |

The descriptions above are call-site roles, not independent behavioral specifications for the dependencies.

## Acceptance examples

These examples are conditional static acceptance criteria, not reports of executed tests.

### Unavailable context

Given readable initial state and F-9b3ea8470521 returning zero:

- The state’s first word is read before that call.
- No record-array read, lookup, formatting call, operation call, or final marker call occurs.
- The function returns without an explicit result.

### Missing object

Given a nonzero context, valid selected record, and lookup returning zero:

- The creation formatting call receives a reread of the selected record word.
- The identifier is read from state byte offset 328 after that call.
- The common tail uses that identifier.
- The final marker is 1.
- Neither existing-object-only dependency is called, regardless of whether the filter is zero.

### Existing object without a filter

Given a nonzero context, a valid nonzero lookup result, and a zero filter:

- The identifier is read before selector 94 is submitted.
- F-887ba2404a49 receives that identifier, the database selector, and zero.
- F-a836e5479ebf and the selector-9 call use the same captured identifier.
- No deletion-formatting call occurs.
- The final marker is 0.

### Existing object with a filter

Given a nonzero context, a valid nonzero lookup result, and a nonzero filter:

- The deletion-formatting call precedes the identifier read.
- The filter’s original 32-bit value is passed to that dependency.
- No selector-94 call occurs.
- F-a836e5479ebf precedes the common tail.
- The final marker is 0.

If the formatting dependency validly changes the identifier from 17 to 23 before returning, the later calls use 23, not 17.

### Record mutation during lookup

If lookup changes the first word of the selected record from one valid value to another, the creation or filtered formatting call receives the new word because the function rereads that location. It does not necessarily receive the value originally supplied to lookup.

### Unchecked creation failure

If the creation formatting dependency returns normally without updating state byte offset 328, the function still reads that word and continues with its current contents. No local failure branch is established.

## Uncertainties

The following are **documented_only**:

- Complete creation and deletion format strings.
- Semantic names for selectors 94, 98, and 9.
- Whether calls generate deferred operations or perform immediate database changes.
- The meaning and validity range of the operation operand.
- Whether the filter operand denotes text, an identifier, or another representation.
- Whether the selected-record word denotes a database name or another handle.
- Error conventions, nonlocal exits, and dependency return meanings beyond the two tested zero/nonzero results.
- Dependency preservation of globals, referenced objects, and object lifetimes.
- Ownership and release obligations.
- Overflow behavior outside valid indexing preconditions.
- Thread safety, synchronization, reentrancy, and transactional guarantees.

Inferred intent is preparation of statistics-table work, with different handling for creation, an unfiltered existing object, and a filtered existing object. That inference does not replace the narrower established call and read contract.

## Evidence

This contract is based solely on the supplied static function representation and the provided opaque dependency mapping.

Established evidence includes 32-bit accesses and operands, offsets 0, 8, 20, and 328, a 24-byte record stride, branch conditions, visible string fragments, literal selectors, call arguments, and their ordering.

No dependency implementation, complete format-string contents, runtime trace, or external database behavior was supplied. No runtime tests have been performed.
