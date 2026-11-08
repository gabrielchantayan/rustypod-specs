<!-- Trusted provenance: F-66eadadc912e; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-66eadadc912e

## Purpose

F-66eadadc912e selects one of four fixed-stride records for a value constructed from its inputs and additional context. It first searches for a record whose existing value compares equal to the constructed value. Only if all four comparisons fail does it search for a record accepted by a separate eligibility dependency.

On either successful path, it increments a 32-bit field in the selected record, writes the selected index to the caller’s output location, and returns zero. On the second path, it also invokes an assignment dependency unless the selected record’s address equals the temporary object’s address.

The apparent intent is to acquire or reuse a slot and increase a usage count. That interpretation is **inferred intent**, not an established ownership or concurrency contract. In particular, the supplied evidence does not establish that the incremented field is a reference count or that the apparent synchronization dependencies provide mutual exclusion.

## Interface

### Inputs and result

The visible interface has three inputs:

| Input | Visible representation | Use |
|---|---|---|
| Selector | One-byte character value | Zero selects an early rejection path. Any nonzero value is passed to the construction dependency. |
| Payload | Opaque 32-bit value | Forwarded to the construction dependency without visible interpretation. |
| Index output | Pointer to a 32-bit integer | Written once on a normal successful path with an index from 0 through 3. Not directly read. |

The result is a 32-bit value with these established normal-return outcomes:

- **0:** a record was selected, its counter field was incremented, and its index was written.
- **0x15:** the selector was zero.
- **0x19:** the selector was nonzero, all first-pass comparisons returned nonzero, and all second-pass eligibility checks returned nonzero.

These are the wrapper’s explicit result values. The evidence does not establish a broader status-code vocabulary or meanings such as “invalid argument” or “capacity exhausted.”

The selector’s signed interpretation is not established. Only its equality to zero matters to the visible branch; the original nonzero byte is forwarded to the construction dependency.

### Record layout

An initial dependency supplies a base address. The wrapper treats that address as the start of four records, each **0x24 bytes, or 36 bytes**, apart.

The following locations are significant within each record:

| Byte offset | Visible use |
|---|---|
| 0x00 | Record address passed to the eligibility and assignment dependencies. |
| 0x04 | A 32-bit value loaded and supplied as the first comparison argument. |
| 0x14 | A 32-bit field read, incremented by one, and written on success. |
| 0x1c | Address passed to the two dependencies that bracket processing of a record. |

The wrapper does not itself establish the types or sizes of objects beginning at these offsets. In particular, the bracketing dependencies receive an address at offset 0x1c, not a value loaded from that address.

### Other context and temporary storage

The nonzero-selector path uses context addresses returned by F-62673223987f:

- One context contributes a byte read at offset **0x254**.
- A second context contributes an address at offset **0x24c**, passed to F-b9acea4b06ad.

These are separate dependency invocations. The second has no explicit argument in the supplied evidence. It is not established that they return the same context.

The wrapper also provides:

- A **20-byte local region** to F-b9acea4b06ad, later passed to F-b5f4a3351c2f and F-289ec4311c64.
- A second temporary region whose start is passed to construction, extraction, and cleanup dependencies.
- A **32-bit word four bytes after that second temporary’s start**, supplied to F-151ab4bfe553 on the assignment path.

The second temporary’s complete object layout and initialized extent are unknown. The evidence shows these accesses but does not establish an eight-byte public object type.

### Preconditions

For normal completion, all dependencies must return normally and satisfy their own memory and argument requirements.

For a successful path:

- The base address must support the required record accesses and dependency operations.
- The selected counter field must support a 32-bit read and write.
- The output pointer must support a 32-bit write.

The wrapper does not validate these conditions. A null output pointer is not directly dereferenced on either explicit failure path, but this does not prove that such a call is safe: dependencies run first and have unknown requirements and effects.

## Behavior

### Entry and selector handling

F-6e90ba5205cc is called before the selector is tested. Its result becomes the record base used by the nonzero-selector path.

If the selector is zero, the wrapper returns **0x15** immediately after that call. It does not visibly construct temporary objects, scan records, increment a counter, or write the output index.

Thus, zero-selector rejection is not a dependency-free operation.

### Construction for a nonzero selector

The wrapper performs these observable operations in order:

1. Calls F-62673223987f with the record base as its explicit argument.
2. Reads one byte at offset 0x254 from that call’s returned address.
3. Calls F-62673223987f again, with no explicit argument shown.
4. Passes the 20-byte temporary’s address and the second returned address plus 0x24c to F-b9acea4b06ad.
5. Calls F-b5f4a3351c2f with the second temporary’s address, selector, 20-byte temporary’s address, previously read byte, and payload.
6. Calls F-289ec4311c64 on the 20-byte temporary.

The wrapper does not inspect a construction status or branch on a return from these preparation operations. Whether construction can fail internally, signal an error elsewhere, or terminate control flow is unknown.

### First pass: comparison against existing records

The first pass visits indices **0, 1, 2, 3**, stopping at the first comparison result equal to zero.

For each visited record:

1. F-338d16f9ce72 receives the record address plus 0x1c.
2. F-44362605ab79 receives the constructed temporary’s address.
3. The wrapper loads the 32-bit value at record offset 0x04.
4. F-4a6c0bf0740c receives that loaded value and the result from F-44362605ab79.
5. If comparison returns nonzero, F-1333331b2da1 receives the same offset-0x1c address before the next record is visited.

The load at offset 0x04 occurs **after** F-44362605ab79 returns. The extraction call is repeated for every visited record; the wrapper does not extract once and cache its result.

If comparison returns zero, the wrapper immediately takes the common success path. It does not call the second-pass eligibility dependency or assignment dependency for that selection.

This establishes a global two-pass priority: even a match at index 3 is selected before an otherwise eligible record at index 0 is considered for assignment.

### Second pass: eligibility and assignment

The second pass starts only after four nonzero comparison results. It again visits indices **0, 1, 2, 3**.

For each visited record:

1. F-338d16f9ce72 receives the record address plus 0x1c.
2. F-96d4ea8519eb receives the record’s starting address.
3. If the eligibility result is nonzero, F-1333331b2da1 receives the offset-0x1c address, and scanning continues.

A zero eligibility result selects the record.

Before incrementing its counter, the wrapper compares the selected record’s starting address with the constructed temporary’s starting address:

- If the addresses differ, F-151ab4bfe553 receives the selected record address and the 32-bit word located four bytes into the temporary.
- If the addresses are equal, that dependency is skipped.

The identity guard is an observed behavior. Its presumed role as self-assignment protection is inferred.

The wrapper does not inspect an assignment result. Any mutation performed by the assignment dependency precedes the counter read used for the success increment.

### Common success path

For a record selected by either pass, the wrapper performs these operations in order:

1. Reads the selected record’s 32-bit field at offset 0x14.
2. Adds one and writes the result back to that field.
3. Writes the selected index as a 32-bit integer through the output pointer.
4. Calls F-1333331b2da1 with the selected record address plus 0x1c.
5. Calls F-289ec4311c64 on the constructed temporary.
6. Returns **0**.

There is no visible saturation check, overflow report, or rollback. The supplied representation uses a 32-bit integer field; the intended signedness and boundary semantics are not established. A mathematical, unbounded increment must not be assumed for boundary values.

The index is written before the final bracketing call and temporary cleanup. Those dependencies cannot be assumed to preserve the output location, selected record, or global state.

### Exhaustion

If all four eligibility results are nonzero:

1. Each visited record receives its corresponding F-1333331b2da1 call.
2. F-289ec4311c64 receives the constructed temporary.
3. The wrapper returns **0x19**.

On this path, the wrapper performs no direct output write and no direct counter increment. Dependency effects remain possible.

## Side effects

### Direct memory effects

The wrapper directly reads:

- The byte at the first context’s offset 0x254.
- A 32-bit field at offset 0x04 for each first-pass comparison.
- The temporary’s offset-4 word when the assignment call is taken.
- The selected record’s counter field on success.

It directly writes, on success only:

- The incremented 32-bit field at record offset 0x14.
- The selected 32-bit index through the output pointer.

No alias restrictions are shown. If the output overlaps a record field or another dependency-visible object, the output write follows the counter write and precedes final dependency calls. Such overlap can change the resulting state; independent output and record storage is a necessary assumption for ordinary use.

### Lifetime and ownership

The local temporary addresses are passed to dependencies. The wrapper visibly invokes F-289ec4311c64:

- On the 20-byte temporary after construction of the other temporary.
- On the constructed temporary before either normal nonzero-selector return.

These calls suggest lifecycle cleanup, but do not establish destruction, deallocation, ownership transfer, or reference management. Likewise, passing the temporary’s offset-4 word into F-151ab4bfe553 does not establish whether it is copied, retained, consumed, or merely inspected.

No direct heap allocation or release is shown. Dependencies may perform either.

### Concurrency and partial effects

The paired offset-0x1c calls suggest per-record synchronization. Their actual semantics are unknown. There is no visible wrapper-level operation protecting the entire two-pass search.

Even if those calls acquire and release locks, the search is not a single protected snapshot: records are processed separately, and the second pass revisits them.

No recovery path is shown for dependency failure, invalid memory, or interrupted execution. A counter write or output write may already have occurred before a later dependency prevents normal return.

## Dependencies

| Opaque function ID | Established use |
|---|---|
| F-6e90ba5205cc | Supplies the record base; called on every invocation. |
| F-62673223987f | Supplies two context addresses on the nonzero-selector path; only the first call has an explicit argument shown. |
| F-b9acea4b06ad | Receives the 20-byte temporary and a context-derived address. |
| F-b5f4a3351c2f | Receives the constructed temporary, selector, intermediate temporary, context byte, and payload. |
| F-289ec4311c64 | Receives each temporary at its visible cleanup point. |
| F-338d16f9ce72 | Receives each visited record’s offset-0x1c address before record processing. |
| F-1333331b2da1 | Receives the same address after rejection or after successful state and output writes. |
| F-44362605ab79 | Produces a comparison argument from the constructed temporary, once per first-pass record. |
| F-4a6c0bf0740c | Compares a loaded record value with the extracted argument; zero selects the record. |
| F-96d4ea8519eb | Determines second-pass eligibility; zero selects the record. |
| F-151ab4bfe553 | Receives a selected record and the temporary’s offset-4 word before second-pass success. |

No dependency is assumed pure, non-allocating, globally preserving, or ownership-neutral.

## Acceptance examples

These are **conditional acceptance scenarios**, not executed tests. Unless stated otherwise, assume valid disjoint storage, normal dependency returns, and no dependency mutations beyond those specified.

1. **Zero selector.**  
   F-6e90ba5205cc is called once. The result is 0x15. No other listed dependency is called by the wrapper, and the output is not directly written.

2. **First record matches.**  
   The first comparison returns zero, and the selected counter initially contains 7. The counter becomes 8, the output becomes 0, and the result is zero. No eligibility or assignment call occurs.

3. **Late match takes precedence over early eligibility.**  
   Comparisons at indices 0–2 return nonzero; index 3 returns zero. Index 3 is selected even if index 0 would have passed eligibility. Eligibility is never queried.

4. **First-pass miss, second-pass selection.**  
   All comparisons return nonzero. Eligibility returns nonzero for index 0 and zero for index 1. If its address differs from the temporary’s address, index 1 receives the assignment call before its counter is incremented. The output becomes 1 and the result is zero.

5. **All records rejected.**  
   All four comparisons and all four eligibility calls return nonzero. The constructed temporary receives cleanup, the result is 0x19, and the wrapper leaves a preexisting output value directly untouched.

6. **Assignment changes the counter.**  
   The second pass selects index 2. Its assignment dependency changes the counter to 12. The wrapper subsequently increments that value to 13, rather than incrementing a value captured before assignment.

7. **Extraction changes a stored comparison field.**  
   During a first-pass visit, F-44362605ab79 changes that record’s offset-0x04 field. The comparison receives the subsequently loaded value, not a value captured before extraction.

## Uncertainties

The following remain **documented_only**:

- The second context call’s complete calling convention and implicit inputs.
- Whether both context calls identify the same object.
- Temporary-object layouts, initialization guarantees, and ownership.
- Comparison semantics beyond zero versus nonzero.
- The property represented by second-pass eligibility.
- Synchronization, reentrancy, and memory-ordering guarantees.
- The counter’s semantic meaning and overflow behavior.
- Dependency error signaling and abnormal-return behavior.
- Effects of dependencies on globals, aliased storage, and retained addresses.
- Valid selector and payload domains beyond the visible zero-selector check.

“Reuse,” “assignment,” “cleanup,” “counter,” and “synchronization” describe visible roles or inferred intent, not independently established dependency contracts.

## Evidence

This contract is based solely on the supplied static function representation and opaque dependency mapping. Established evidence includes the branch conditions, explicit status values, four-record scan bounds, record stride and offsets, dependency argument flow, and ordering of direct memory accesses.

No implementation of a dependency was supplied. No runtime tests, traces, concurrency checks, or ownership experiments have been performed. The acceptance examples are predictions conditional on their stated assumptions, not observed results.
