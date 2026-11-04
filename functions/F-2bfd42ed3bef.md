<!-- Trusted provenance: F-2bfd42ed3bef; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-2bfd42ed3bef

## Purpose

F-2bfd42ed3bef produces a signed 32-bit result from a tagged input object and a signed 32-bit query value.

Two input tags have defined processing paths:

- **Tag 1:** initialize an opaque traversal, count how many traversal calls return a nonzero status, invoke a final dependency, and return the count.
- **Tag 2:** search a sequence of fixed-stride records and return the value associated with the first matching key.

Other tags return zero without invoking the listed dependencies.

These are descriptions of the established control flow, not claims about the application-level meaning of the object. In particular, zero is not an unambiguous “not found” indicator: it can also represent an empty traversal, a matching record whose value is zero, or an unsupported tag.

## Interface

### Inputs

The function accepts two arguments:

1. **Object address:** an address from which the function reads a one-byte discriminator and, for recognized tags, a four-byte field at byte offset 4.
2. **Query:** a signed 32-bit integer. The function forwards it to the initialization dependency for tag 1 and compares it against record keys for tag 2.

No object size, record-storage size, or output-buffer argument is supplied. Validity and sufficient extent of all accessed storage are caller preconditions; the function performs no explicit validation.

### Input object layout

| Byte offset | Width | Established use |
|---|---:|---|
| 0 | 1 byte | Tag, compared against 1 and 2 |
| 1–3 | 3 bytes | No direct access shown |
| 4 | 4 bytes | Opaque argument for tag 1; address-bearing field for tag 2 |

For tag 1, the field at offset 4 is passed as a four-byte value. Its semantic type is unknown.

For tag 2, the same field is interpreted as the address of a secondary structure:

| Secondary-structure offset | Width | Established use |
|---|---:|---|
| 12 | 4 bytes | Unsigned 32-bit record count |
| 16 | 4 bytes | Address of the first record |

Other secondary-structure fields are not directly read by this function.

The record stride is 16 bytes:

| Record-relative offset | Width | Established use |
|---|---:|---|
| 0 | 4 bytes | Signed 32-bit key |
| 4 | 4 bytes | Signed 32-bit result value |
| 8–15 | 8 bytes | Not directly read |

This contract follows the supplied representation’s four-byte integer and address-bearing accesses. Target byte order, alignment requirements, and address-space rules are not independently established.

### Output

The result is represented as a signed 32-bit integer:

- For tag 1, the number of nonzero traversal statuses, subject to the arithmetic limitation described below.
- For tag 2, the value in the first matching record, or zero if no match is reached.
- For other tags, zero.

There is no separate status result, diagnostic output, or explicit error channel.

## Behavior

### Tag selection and read order

The logical result starts at zero.

The function first reads the byte at object offset 0 and compares it with 1. If that comparison fails, the supplied representation evaluates another read of the tag byte and compares it with 2.

For stable input, this selects the expected tag-1, tag-2, or unsupported-tag path. The two comparisons are not presented as using one saved tag value. Exact machine-load behavior under concurrent mutation or special memory semantics is not established by this representation.

A readable first byte is required even for unsupported tags. For stable unsupported tags, the field at offset 4 is not read.

### Tag 1: count an opaque traversal

The tag-1 path has the following ordered interactions.

1. Read the four-byte object field at offset 4.
2. Invoke **F-99bb1fbf801b**, passing:
   - the address of a four-byte local scratch region;
   - the four-byte value read from the object;
   - the query.
3. Invoke **F-37f0845bb2ad** repeatedly, passing the same three scratch addresses on every invocation:
   - the four-byte region supplied to initialization;
   - a second four-byte local region;
   - a third four-byte local region.
4. After each nonzero traversal status, increment the signed 32-bit count by one.
5. On the first zero traversal status, stop invoking the traversal dependency.
6. Invoke **F-944b595e25e9**, passing the address of a distinct 20-byte local scratch region.
7. Return the count, provided the dependencies return normally and no invalid operation intervenes.

The caller does not explicitly initialize the scratch regions before handing their addresses to dependencies. Any required initialization must therefore come from dependency behavior or another contract not supplied here.

The traversal is called at least once after initialization returns normally. An immediately zero status produces a count of zero. All nonzero statuses count equally, including negative values; no special error interpretation is visible.

The outputs written through the second and third traversal arguments are not examined directly by this function. They may still affect subsequent calls if the dependency reads their retained contents. The function neither clears nor resets these regions between calls.

The final dependency receives a different address from all three traversal arguments. The supplied representation places its 20-byte region immediately after the first four-byte region, but that layout does not establish a common object type, ownership relationship, or destructor relationship.

The function contains no independent iteration bound. If traversal never returns zero, normal completion is not established. The final dependency is reached only after a zero traversal status; it is not shown as an unconditional exit handler.

### Tag 1 arithmetic boundary

The count is represented as a signed 32-bit integer, begins at zero, and is incremented once per nonzero traversal return.

Counts from 0 through 2,147,483,647 are representable. A further increment exceeds that signed range. The supplied representation does not establish whether the actual executable wraps, traps, or has another consequence at that boundary. No overflow check, saturation rule, wider accumulator, or overflow error is shown.

Accordingly, an exact numerical return is established only when the count remains representable and all invoked dependencies complete normally.

### Tag 2: first-match record lookup

The tag-2 path first obtains the record cursor:

1. Read the address-bearing field at object offset 4.
2. Read the four-byte record-array address at offset 16 of that secondary structure.
3. Retain that address as the initial record cursor.

This happens before the first count comparison. Thus, even a zero-count lookup requires readable storage for the secondary structure’s record-array field. It does not require a record-key read merely because the count is zero.

The unsigned 32-bit loop index starts at zero. Before each candidate record:

1. Read the secondary-structure address again from object offset 4.
2. Read the unsigned 32-bit count at offset 12 of that structure.
3. Compare the current index against that count.

When the index is not less than the count, the function returns zero.

When the index is less than the count, the function reads the signed 32-bit key at the current record cursor:

- If the key equals the query, read the signed 32-bit value four bytes after that key and return it immediately.
- If the key differs, advance the record cursor by 16 bytes, increment the unsigned index by one, and perform the next count comparison.

Consequences include:

- Matching uses exact 32-bit integer equality, not ordering or a partial-key comparison.
- Duplicate keys resolve to the earliest matching record in traversal order.
- A matching value is returned unchanged, including zero or a negative value.
- A nonmatching record’s associated value is not directly read.
- Bytes 8–15 of each record are not directly read.
- No later record is examined after a match.
- The record-array address is captured once; the secondary-structure address and count are read again for successive loop conditions.

For stable input and a count of \(N\), at most \(N\) keys are examined. If all miss, the count is evaluated again when the index reaches \(N\), and the function returns zero.

The count is unsigned. A stored count with its high bit set is therefore a large positive bound, not a negative count or a request to skip searching. Such counts require correspondingly valid accessible record storage.

The record cursor advances in 16-byte steps. Valid address arithmetic and sufficient storage are preconditions; no address-overflow or storage-extent check is shown.

## Side effects

### Direct effects

The supplied function directly:

- reads the tag and recognized-tag payload field;
- reads secondary metadata and record fields on tag 2;
- updates its local count, index, cursor, and scratch storage as described;
- invokes three dependencies on tag 1.

It contains no explicit write through the input object address or into the tag-2 record storage.

### Dependency-mediated effects

Dependency effects are unknown. The function must not be treated as pure merely because its own input accesses are reads.

In particular, the supplied evidence does not establish that dependencies:

- preserve global state;
- preserve externally reachable objects;
- allocate or release memory;
- leave the input object unchanged;
- return normally;
- refrain from retaining scratch addresses.

No ownership transfer is explicitly established. The scratch regions are local to this invocation; their availability after return is not established. A dependency that retains their addresses would require a lifetime guarantee absent from the supplied evidence.

There is no explicit allocation or deallocation in the caller. The final dependency’s position suggests finalization intent, but does not prove resource release or identify any resource owner.

### Failures and abnormal completion

The function has no explicit checks for null addresses, inaccessible memory, insufficient storage, invalid embedded addresses, or misalignment.

No normal return value is specified here for a memory fault, dependency non-return, dependency-induced invalid state, or unestablished arithmetic behavior. Zero must not be interpreted as a general recovery result for those conditions.

## Dependencies

| Opaque ID | Invocation | Established use |
|---|---|---|
| **F-99bb1fbf801b** | Once on tag 1, before traversal | Receives a local four-byte scratch address, the object’s four-byte payload field, and the query. No return value is used. |
| **F-37f0845bb2ad** | Repeatedly on tag 1 | Receives three local four-byte scratch addresses. Its integer status controls continuation and counting. |
| **F-944b595e25e9** | Once after traversal returns zero | Receives a distinct local 20-byte scratch address. No return value is used. |

There are no dependency calls on the stable tag-2 or unsupported-tag paths.

The labels “initialization,” “traversal,” and “finalization” describe call placement and inferred roles. They are not independently verified dependency contracts.

## Acceptance examples

These examples are conditional specifications derived from static evidence. They are not runtime test results. Each assumes valid memory, stable input where relevant, normal dependency completion, and representable arithmetic.

### Unsupported tag

Given a readable tag byte equal to 0, 3, or 255:

- The result is zero.
- No listed dependency is invoked.
- No read of the four-byte field at offset 4 is required by this path.

### Empty traversal

Given tag 1 and a traversal dependency that returns zero on its first invocation:

- Initialization occurs once.
- Traversal occurs once.
- The final dependency occurs once after traversal.
- The result is zero.

### Nonzero traversal statuses

Given tag 1 and successive traversal statuses of 1, −7, 2, and 0:

- Traversal occurs four times.
- The three nonzero statuses each increment the count.
- The final dependency occurs after the fourth traversal call.
- The result is 3.

The negative status is not treated as an error by this function.

### Zero-count lookup

Given tag 2, readable secondary metadata, and an unsigned count of zero:

- The record-array address field is still read.
- No record key or associated value is read.
- The result is zero.

An empty lookup does not, by itself, require the record-array address to designate a readable record.

### First matching record

Given three records with key/value pairs \((8, 40)\), \((12, -3)\), and \((12, 91)\), and query 12:

- The first key is examined and rejected.
- The second key matches.
- The result is −3.
- The third record is not examined.

### Matching zero value

Given a matching record whose associated value is zero:

- The function returns zero immediately at that record.
- The return value alone cannot distinguish this match from a miss.

### No match

Given a stable count of two and two readable records whose keys both differ from the query:

- Both keys are examined in sequence.
- Neither associated value needs to be read.
- The result is zero after the terminating count comparison.

### Maximum representable traversal count

If traversal returns nonzero exactly 2,147,483,647 times and then zero, the count remains representable and the specified return is 2,147,483,647.

If another nonzero status occurs instead, a portable exact return is not established by the supplied evidence.

## Uncertainties

The following remain **documented_only**:

- **Semantic object types:** application-level meanings of tags, payloads, keys, and values are unknown.
- **Dependency contracts:** initialization rules, scratch layouts, output meanings, global effects, allocation behavior, and failure behavior are unavailable.
- **Finalization target:** the final dependency’s argument is distinct from the initialization argument. Its relationship to traversal state is not established.
- **Signed overflow:** behavior beyond the representable traversal count is unknown.
- **Machine details:** byte order, strict alignment requirements, address validity rules, and exact instruction-level read behavior are not independently verified.
- **Concurrent mutation:** repeated metadata accesses are visible in the supplied representation, but no synchronization or concurrent-access contract is supplied. Coherent behavior under racing mutation must not be assumed.
- **Memory ownership:** nothing establishes that this function acquires ownership of the input, secondary structure, or records, or that a dependency releases them.
- **Abnormal exits:** cleanup and a return value are not guaranteed if a dependency fails to return normally.

## Evidence

**Established static evidence:** The supplied function representation shows tag comparisons against 1 and 2; four-byte payload and metadata accesses; a tag-1 dependency sequence with nonzero-status counting; and a tag-2 first-match search using an unsigned count, 16-byte record stride, and four-byte key/value fields.

**Inferred intent:** Tag 1 appears to count items exposed by an opaque traversal. Tag 2 appears to implement a key-to-value lookup. The final tag-1 call appears related to finalization. These interpretations do not add guarantees about dependency semantics or ownership.

**Verification limit:** No runtime tests have been performed. Acceptance examples are conditional consequences of the supplied static representation, not observed executions.
