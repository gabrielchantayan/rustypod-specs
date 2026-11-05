<!-- Trusted provenance: F-2b6fe108bb2f; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-2b6fe108bb2f

## Purpose

F-2b6fe108bb2f obtains an object address, partially initializes the object, invokes dependencies for three supplied handles, installs those handles into the object, and writes a 32-bit value derived from fields read through the handles.

The derived value is the unsigned maximum of the three observed fields plus one, reduced modulo \(2^{32}\). These fields are read after the handle-processing calls and after the handles are installed; their entry values do not necessarily determine the result.

On normal completion, the function returns the address obtained from its first dependency. The final dependency may change the object before that return.

Construction, retention, registration, and finalization are possible interpretations, not established semantics. The contract establishes memory accesses, arithmetic, call arguments, ordering, and the returned address without assigning undocumented ownership behavior to dependencies.

## Interface

### Inputs

The function has four parameter positions:

| Position | Established use |
|---|---|
| First | A 32-bit value unused by the visible body. It is neither inspected nor forwarded. |
| Second | A 32-bit handle passed to a dependency, stored at object byte offset 32, and used as an address for a 32-bit read at byte offset 8. |
| Third | A 32-bit handle passed to a dependency, stored at object byte offset 24, and used as an address for a 32-bit read at byte offset 8. |
| Fourth | A 32-bit handle passed to a dependency, stored at object byte offset 28, and used as an address for a 32-bit read at byte offset 8. |

The supplied representation treats the three handles as signed integer parameters. Their field contents, however, are compared as unsigned 32-bit quantities. The evidence does not establish a semantic requirement that handle values be positive.

No visible check rejects a zero handle, duplicate handles, or overlapping addressed storage.

### Output

The normal return value is the same object address returned by F-3ec664b2ca65.

There is no explicit failure return, status result, or null-address check. The function begins writing through the returned address before invoking the next dependency.

Returning that address does not establish ownership or guarantee that the object remains live after the final dependency.

### Directly accessed object layout

All offsets are byte offsets from the obtained object address. Each listed field is 32 bits wide.

| Offset | Direct accesses |
|---|---|
| 0 | Written once with a value read from a global header-value location. |
| 8 | Written once with the unsigned maximum-plus-one result. |
| 24 | Cleared, conditionally inspected, then overwritten with the third parameter. |
| 28 | Cleared, conditionally inspected, then overwritten with the fourth parameter. |
| 32 | Cleared, conditionally inspected, then overwritten with the second parameter. |

The body directly writes **five distinct fields through eight stores**. Offsets 24, 28, and 32 each receive two stores; offsets 0 and 8 each receive one.

The highest directly accessed object field ends at byte offset 35. Normal execution therefore requires storage supporting these accesses, but the complete object size and the meanings of other bytes are unknown. Partial initialization must not be treated as whole-object initialization.

### Preconditions for normal completion

Normal completion requires:

- F-3ec664b2ca65 to return an address supporting subsequent object accesses.
- The captured context-cell address to support both later 32-bit reads.
- Each handle-derived address to support its offset-8 read when that read occurs.
- Every invoked dependency to return normally.
- Storage needed by later operations to remain accessible despite intervening calls and writes.

These conditions are not checked by the visible body. Alignment, address calculation validity, memory mapping, and fault behavior are not fully specified by the supplied evidence.

## Behavior

### Object acquisition and context capture

The first operation is a call to F-3ec664b2ca65 with no explicit arguments. Its result is retained as the object address used throughout the function and returned on normal completion.

After this call returns, the function reads a global pointer to a context cell and retains that address.

The timing matters: the initial dependency may affect the global pointer or the global header value before either is read. No preservation guarantee is available.

The context-cell address is captured once, but its contents are read twice. Later changes to the global pointer alone do not redirect either use of the captured address. Changes to the captured cell itself can change the value passed to the final dependency.

### Partial initialization

After capturing the context-cell address, the body performs the following direct accesses in order:

1. Read the global header value and store it at object offset 0.
2. Store zero at object offset 24.
3. Store zero at object offset 28.
4. Store zero at object offset 32.

The context-pointer global and header-value global are separate locations. Their concrete values are not supplied.

Object offset 8 has not yet been assigned by this function. No other object bytes are visibly initialized.

### First context/object call

The function reads a 32-bit value from the captured context cell and calls F-03b9616af95f with that value followed by the object address.

At this call boundary, the body has performed the header store and the three clearing stores. This describes completed writes, not a guarantee that every written value remains unchanged under aliasing or dependency effects.

The dependency’s behavior is unavailable. In particular, it may alter object fields subsequently inspected by this function.

### Handle-processing calls

The function calls F-2609ef4ae406 three times:

1. With the second parameter.
2. With the third parameter.
3. With the fourth parameter.

Each call has one explicit argument. No returned value is used by the visible body.

All three calls occur without a preceding nonzero check. Equal handles still produce separate calls. The calls precede both the handle-installation stores and the handle offset-8 reads, so any changes they make to subsequently accessed storage can affect later behavior.

The evidence does not establish that these calls retain, validate, or otherwise manage the handles.

### Sequential conditional inspections

Next, the function reads object fields in this order:

| Inspection | Object offset | Action when the observed value is nonzero |
|---|---|---|
| First | 32 | Call F-029930293dcb with no explicit arguments. |
| Second | 24 | Call F-029930293dcb with no explicit arguments. |
| Third | 28 | Call F-029930293dcb with no explicit arguments. |

Each field is read at its own inspection point. The function does not first take a snapshot of all three fields.

Consequently, a helper call triggered by offset 32 can change what is later observed at offsets 24 and 28. A call triggered by offset 24 can change the later observation at offset 28.

The inspected value is not explicitly passed to F-029930293dcb. A resource-release interpretation is therefore unestablished.

The earlier clearing stores do not make these conditions irrelevant: intervening dependencies may have modified the fields.

### Handle installation

After the inspections and any resulting calls, the function stores:

1. The third parameter at object offset 24.
2. The fourth parameter at object offset 28.
3. The second parameter at object offset 32.

No dependency call occurs between these three stores. They overwrite the fields’ then-current contents without additional checks or visible preservation of the overwritten values.

### Input-field observations

The function then performs three 32-bit reads in this order:

1. The third handle’s addressed storage at byte offset 8.
2. The fourth handle’s addressed storage at byte offset 8.
3. The second handle’s addressed storage at byte offset 8.

All three reads occur before the result-field store.

These are late observations. Earlier dependencies, conditional helper calls, and handle-installation stores may have changed the addressed bytes. This is particularly relevant when handles alias the object, globals, the captured context cell, or each other.

Distinct parameter positions do not imply distinct storage. Equal addresses do not eliminate any of the three specified reads.

### Unsigned result calculation

Let \(A\), \(B\), and \(C\) denote the 32-bit values just observed through the second, third, and fourth handles, respectively.

The function compares these values as unsigned 32-bit quantities and writes the following 32-bit result at object offset 8:

\[
R = \bigl(\max(A,B,C)+1\bigr)\bmod 2^{32}.
\]

The comparison structure selects the unsigned maximum even when two or three values are equal.

The increment does not saturate and does not produce a wider stored result. In particular:

- Three zero observations produce 1.
- An observation with its high bit set is treated as an unsigned value of at least \(2^{31}\).
- Any observation equal to \(2^{32}-1\) causes the stored result to be zero.

The modulo statement describes the 32-bit result arithmetic. It does not establish rules for invalid handle-address calculations.

### Final context/object call and return

After storing the result, the function reads the 32-bit contents of the previously captured context cell again.

It calls F-e31e208b4293 with that freshly read value followed by the object address, then returns the original object address.

No direct object-field write follows this call. The final dependency may change any previously written field; the body does not restore those fields before returning.

## Side effects

The body performs **eight direct 32-bit stores to five distinct object fields**:

| Offset | Store count | Values, in order |
|---|---:|---|
| 0 | 1 | Global header value |
| 8 | 1 | Unsigned maximum plus one, modulo \(2^{32}\) |
| 24 | 2 | Zero; third parameter |
| 28 | 2 | Zero; fourth parameter |
| 32 | 2 | Zero; second parameter |

Its direct reads include:

- One read of the global context-cell pointer.
- One read of the global header value.
- Two reads of the captured context cell.
- Three sequential object-field inspections.
- Three handle offset-8 reads.

On a normally completed execution, the dependency order is:

1. F-3ec664b2ca65.
2. F-03b9616af95f.
3. F-2609ef4ae406, three times.
4. F-029930293dcb, zero to three times according to the sequential inspections.
5. F-e31e208b4293.

There are six unconditional calls and zero to three conditional calls, for a total of six through nine.

These counts describe operations in this function’s body, not additional accesses or calls inside dependencies.

No direct deallocation, explicit error report, rollback, or compensating cleanup is visible. If a dependency fails to return or an access cannot complete, the evidence provides no recovery behavior.

Ownership transfer, retention, release, and continued object lifetime are not established.

## Dependencies

| Opaque function ID | Visible interface use | Unresolved behavior |
|---|---|---|
| F-3ec664b2ca65 | No explicit arguments; result used as the object address. | Allocation, initialization, failure behavior, ownership, and lifetime. |
| F-03b9616af95f | Receives a context-cell value and the partially initialized object address. | Mutations, registration, ownership effects, and failure behavior. |
| F-2609ef4ae406 | Receives each supplied handle in second/third/fourth parameter order. | Validation, retention, release, mutation, and failure behavior. |
| F-029930293dcb | No explicit arguments; called once per qualifying sequential inspection. | Relationship to the inspected value, resource effects, and mutations. |
| F-e31e208b4293 | Receives a fresh context-cell value and the object address after the result store. | Publication, finalization, mutations, ownership effects, and lifetime. |

No dependency may be assumed to preserve globals, object fields, context-cell contents, or handle-addressed storage. The two context/object calls use the same captured cell address but need not receive the same cell value.

## Acceptance examples

These examples are conditional consequences of static evidence, not executed tests. Unless stated otherwise, assume accessible storage, normally returning dependencies, no relevant dependency mutations, and no interfering aliasing.

### Ordinary values and store accounting

If the second, third, and fourth handles expose offset-8 values of 5, 12, and 9 at their respective reads, object offset 8 receives 13.

The body performs eight direct stores across offsets 0, 8, 24, 28, and 32. It does not write a sixth distinct object field.

After installation, offsets 24, 28, and 32 contain the third, fourth, and second handles, respectively, provided later operations do not alter them. The function returns the address obtained from F-3ec664b2ca65.

### Unsigned and wraparound boundaries

If all three observed fields are zero, the result store writes 1.

If the observations are 2,147,483,648, 7, and 9, the result store writes 2,147,483,649. Treating the first observation as negative would not match the unsigned comparison.

If any observed field is 4,294,967,295, the result store writes zero. Saturation, overflow rejection, or storing 4,294,967,296 would not match the established arithmetic.

### Repeated handles

If all three handles have the same address, F-2609ef4ae406 is still called three times with that address.

The address is installed into all three handle slots. Three offset-8 reads still occur in third/fourth/second parameter order. Under the default noninterference assumptions, equal observed values produce that value plus one modulo \(2^{32}\).

### Sequential helper effects

If all inspected fields are zero when read, F-029930293dcb is not called.

If all are nonzero at their respective inspection points, it is called three times.

If offset 32 is nonzero and its resulting helper call clears offsets 24 and 28, only one helper call occurs. Taking an early snapshot and making three calls would not match the required sequence.

### Context changes

If the captured cell contains one value at the first context read and another at the final context read, the context/object dependencies receive those respective values.

If only the global context pointer changes after capture, the final read still uses the original captured address, assuming that address remains accessible.

### Final mutation

If F-e31e208b4293 overwrites object offset 8, the function returns the original object address without restoring the computed value.

Acceptance must distinguish the required result store before this call from the object contents observed after return.

## Uncertainties

The following remain **documented_only**:

- Complete object layout, allocation extent, semantic type, and concrete header value.
- The meaning of the handle offset-8 fields and the derived result.
- Dependency allocation, retention, release, registration, publication, or destruction behavior.
- Whether dependencies preserve any relevant memory or globals.
- Whether the returned object remains live after the final call.
- Allocation failure, invalid-handle behavior, exceptions, nonlocal exits, and memory faults.
- Platform rules governing address arithmetic, alignment, byte order, and invalid addresses.
- Concurrency, synchronization, and externally observable memory-access guarantees.
- Whether the surrounding system permits aliasing among the object, handles, globals, and context cell.

The construction-like lifecycle interpretation is inferred intent. The unsigned maximum-plus-one relationship is established arithmetic, but its domain-specific purpose is unknown.

## Evidence

This contract is based on the supplied static function body and its dependency-to-opaque-ID mapping. Dependency implementations and surrounding callers were not supplied.

Established static evidence supports:

- An unused first parameter.
- Five directly written 32-bit object fields at offsets 0, 8, 24, 28, and 32.
- Eight direct stores, including two each at offsets 24, 28, and 32.
- One captured context-cell address and two separate reads of its contents.
- The dependency-call and memory-access ordering described above.
- Sequential conditional helper calls without explicit arguments.
- Three ordered handle-field reads followed by unsigned maximum-plus-one arithmetic.
- Return of the address obtained from the initial dependency.

No runtime tests have been performed. The acceptance examples express conditional requirements derived from static evidence, not observed execution results.
