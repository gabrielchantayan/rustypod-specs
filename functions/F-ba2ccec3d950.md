<!-- Trusted provenance: F-ba2ccec3d950; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-ba2ccec3d950

## Purpose

F-ba2ccec3d950 conditionally writes a requested unsigned 32-bit value into an object, then selects further operations using that request, one or two queried quantities, and an optional attached object.

Established behavior is a gated update-and-dispatch operation. Interpreting the requested value as a limit, capacity, or target is **inferred intent**, not an established meaning. The calls surrounding stored-value accesses resemble synchronization operations, but their actual guarantees are unknown.

The result is not uniformly Boolean. Most paths return 1. A null attached-object pointer on the primary-dispatch path produces 0. Fallback dispatch returns its dependency’s unrestricted 32-bit result.

## Interface

### Inputs and representation

The function accepts:

- **Object base:** an address carried in a 32-bit integer representation in the supplied source.
- **Requested value:** an unsigned 32-bit value, ranging from 0 through 4,294,967,295.

The visible layout is a 32-bit layout: object pointers, dispatch-table pointers, and dispatch entries occupy four bytes. Porting these offsets to another pointer width is not established.

The object base is not validated. Normal completion requires accessible storage for the fields reached on the selected path, valid embedded regions for dependency calls, and valid indirect-call targets when dispatch occurs. A nonzero gate avoids later accesses but still requires a valid byte read at offset `0x70`.

### Object layout

Offsets are byte offsets from the object base.

| Offset | Width or role | Established use |
|---|---|---|
| `0x48` | Four-byte attached-object pointer | Read and captured before primary dispatch. Fallback contains two separate reads of this field: one for target resolution and one for the receiver argument. |
| `0x70` | One-byte gate | Read at entry. Zero enables further work; any nonzero byte bypasses it. |
| `0xC4` | Embedded-region address | Supplied to F-aa65c4c2948b. The region’s size and internal layout are unknown. |
| `0xEC` | Embedded-region address | Supplied to F-U01 and F-U02 around stored-value accesses. The region’s size and internal layout are unknown. |
| `0x108` | Four-byte unsigned stored value | Read on every gate-open invocation; explicitly written only if the captured value differs from the request. |

A reached stored-value access spans offsets `0x108` through `0x10B`.

An attached object’s first four-byte field supplies its dispatch-table address. Primary dispatch uses the entry at table byte offset `0x24`; fallback uses the entry at table byte offset `0x28`.

For fallback, the attached object used to find the table entry and the object supplied as the receiver are obtained by **separate field reads**. Their equality requires an additional stability assumption.

### Output

The function returns an unsigned 32-bit value:

- **1** for gate bypass, zero-request completion, non-dispatch completion, or a primary indirect result of exactly 1;
- **0** when primary dispatch is required but its captured attached-object pointer is null;
- **the fallback operation’s 32-bit result** when fallback is reached.

Fallback may itself return 0 or 1, so those values do not uniquely identify the path taken. There is no separate error object or explicit rollback mechanism.

## Behavior

### 1. Entry gate

The function first reads the byte at offset `0x70`.

If the byte is nonzero, it returns 1 immediately. It does not access the stored-value field, call a dependency, query the embedded region, or read the attached-object field.

If the byte is zero, execution continues. There is no later gate reread. A subsequent change to the gate does not introduce another gate decision within this invocation.

### 2. Capture the stored value

On a gate-open invocation, the function performs these operations in order:

1. Calls F-U01 with the embedded-region address at offset `0xEC`.
2. Reads the unsigned 32-bit stored value at offset `0x108`.
3. Calls F-U02 with the same embedded-region address.

After F-U02 returns, the function compares the captured value with the request. This determines whether the invocation considers the value **changed**.

The decision uses the captured word, not a new read after F-U02. If that dependency modifies the stored field, the modification does not replace the captured value used for comparison.

### 3. Conditional stored-value write

If the captured value differs from the request, the function:

1. Calls F-U01 with the address at offset `0xEC`.
2. Writes the requested value as a four-byte word at offset `0x108`.
3. Calls F-U02 with the same embedded-region address.

If the values are equal, this second bracketed sequence is omitted.

The observation and conditional write are not established as an atomic compare-and-update operation. Dependency calls separate the original read from the write, and there is no second stored-value read immediately before writing.

The changed decision remains based on the original observation. Later dependency effects do not cause it to be recomputed.

### 4. Zero request

After the stored-value handling, a request of zero causes a call to F-7d54bc11e6c7 with the object base.

The function then returns 1 if the dependency returns normally. This happens whether the captured stored value was already zero or required replacement.

This path does not query F-aa65c4c2948b and does not inspect the attached-object field. The result of F-7d54bc11e6c7 is not used.

### 5. First quantity comparison for a nonzero request

For a nonzero request, the function calls F-aa65c4c2948b with the embedded-region address at offset `0xC4`. Its result is consumed as an unsigned 32-bit quantity.

The request is compared against that quantity using unsigned ordering.

If the request is greater than or equal to the first quantity:

- F-3c18ad612b20 is called with the object base if the changed decision is true.
- That call is omitted if the changed decision is false.
- The function returns 1.

Equality belongs to this branch. A first quantity of zero also selects this branch for every nonzero request.

Neither F-22314fdee3bd nor indirect dispatch is reached on this branch.

### 6. Action and second quantity comparison

If the request is strictly less than the first quantity, the function:

1. Calls F-22314fdee3bd with the object base.
2. Calls F-aa65c4c2948b again with the address at offset `0xC4`.
3. Compares the request with the second unsigned 32-bit result.

The second result is obtained by a distinct call, not by reusing the first quantity.

If the request is greater than or equal to the second quantity, the function returns 1 without reading the attached-object field.

F-3c18ad612b20 is not called on this path, even if the changed decision is true. Its use is confined to the branch selected by the first comparison.

If the request remains strictly less than the second quantity, execution proceeds to attached-object dispatch.

### 7. Primary attached-object dispatch

The function reads and captures the attached-object pointer at offset `0x48`.

If the captured pointer is null, it returns 0. All preceding operations have already occurred, including any stored-value write, both quantity queries, and F-22314fdee3bd. None is rolled back.

If the pointer is nonnull, primary dispatch uses that captured pointer both:

- to read the dispatch-table pointer and resolve the entry at byte offset `0x24`; and
- as the receiver argument.

The requested unsigned 32-bit value is the other argument. A nonnull pointer alone does not establish valid receiver storage, a valid table, or a callable entry.

If F-U03 returns exactly the integer value 1, the function returns 1 and does not perform fallback dispatch.

Every other primary result selects fallback. This includes zero, 2, and bit patterns interpreted as negative under a signed 32-bit representation. The condition is exact equality with 1, not general nonzero truth.

### 8. Fallback dispatch: two independent field reads

Fallback does not reuse the primary receiver. It also does not capture one replacement pointer for both target resolution and argument passing.

The fallback expression contains two separate reads of the attached-object field at offset `0x48`:

- **Target-resolution read:** supplies the object whose first field identifies the dispatch table. The entry at table byte offset `0x28` supplies the fallback target.
- **Receiver-argument read:** supplies the receiver passed to that target.

The supplied expression does **not establish the relative evaluation order** of these two reads. Neither read has a fresh null check. The earlier null check applies only to the pointer captured for primary dispatch.

If the field remains stable throughout fallback evaluation, both reads obtain the same pointer. If stability is not established, this contract does not guarantee that the dispatch target is selected through the same object that is passed as receiver.

Normal completion therefore requires valid target resolution and a receiver acceptable to the selected operation. A null or invalid target-resolution pointer can invalidate the dereferences needed to find the target. A null receiver-argument value is not rejected by this function; whether the selected dependency accepts it is unknown.

F-U04 receives the requested value as its other argument. Its 32-bit result becomes the function’s return value without Boolean normalization.

## Side effects

### Direct memory effects

The only explicit object write in the visible body is the four-byte write at offset `0x108`. It occurs only when the initially captured stored value differs from the request.

That write, when performed, precedes:

- F-7d54bc11e6c7 on the zero-request path;
- every quantity query;
- F-3c18ad612b20 or F-22314fdee3bd;
- any indirect dispatch.

There is no explicit write to the gate or attached-object field. There is no explicit allocation, deallocation, reference-count adjustment, or ownership transfer.

The body does not establish that the stored field still equals the request at return. Dependencies called after the write may modify it. Even a no-write path may leave a changed stored field if a dependency modifies memory after the captured read.

### Dependency effects

No dependency is assumed to preserve object fields, globals, memory ownership, object lifetime, or queried quantities.

In particular, this contract does not assume that F-U01 or F-U02 provides locking, visibility, atomicity, or exclusion. Their visible role is call bracketing; their synchronization semantics remain unknown.

Captured decisions remain based on their earlier observations. The gate is not reread, the stored-value comparison is not recomputed, and a query result is retained for its associated comparison. By contrast, fallback explicitly uses two new attached-field reads whose relative order is unspecified.

### Partial completion and errors

There is no explicit recovery for invalid memory access, invalid indirect dispatch, or abnormal dependency termination. The source does not establish a portable fault, exception, or error-return outcome for those cases.

A returned 0 does not imply absence of side effects. It can arise from the checked null primary pointer after substantial prior work, or from the unrestricted fallback result.

The meaning of fallback return values belongs to that dependency’s contract; this function does not classify them as success or failure.

## Dependencies

Mapped dependencies:

| Function ID | Arguments supplied | Result use and call condition |
|---|---|---|
| F-7d54bc11e6c7 | Object base | Result unused. Called for a gate-open zero request after stored-value handling. |
| F-3c18ad612b20 | Object base | Result unused. Called only when the nonzero request is at least the first quantity and the changed decision is true. |
| F-aa65c4c2948b | Address at object offset `0xC4` | Result interpreted as unsigned 32-bit. Called once for a nonzero request, and again only when the request is below the first result. |
| F-22314fdee3bd | Object base | Result unused. Called between the two queries when the request is below the first quantity. |

Local opaque identifiers for dependencies without mapped identities:

| Function ID | Identity boundary | Arguments and result |
|---|---|---|
| F-U01 | First access-bracketing operation | Receives the address at offset `0xEC`; result unused. |
| F-U02 | Second access-bracketing operation | Receives the address at offset `0xEC`; result unused. |
| F-U03 | Indirect operation at table offset `0x24` | Target and receiver use the captured primary pointer. Receives the request; only exact result 1 prevents fallback. |
| F-U04 | Indirect operation at table offset `0x28` | Target resolution and receiver argument use separate reads of offset `0x48`, with unspecified relative evaluation order and no fresh null checks. Receives the request; its result is returned. |

F-U03 and F-U04 identify dispatch positions, not fixed implementations. No ownership or lifetime guarantee follows from any call listed here.

## Acceptance examples

These are **conditional static expectations**, not executed tests. Unless an example states otherwise, accesses and call targets are valid, dependencies return normally, and there are no additional relevant state changes.

1. **Closed gate.**  
   Gate byte 2, request 9: return 1. No dependency call or stored-value access occurs.

2. **Zero already stored.**  
   Gate zero, captured stored value zero, request zero: one F-U01/read/F-U02 sequence occurs, with no explicit write. F-7d54bc11e6c7 is called, then the function returns 1. No quantity query occurs.

3. **Zero replaces a nonzero value.**  
   Captured stored value 7, request zero: a second F-U01/F-U02 pair brackets a write of zero. F-7d54bc11e6c7 follows, then return 1.

4. **Changed request equals the first quantity.**  
   Captured value 4, request 8, first quantity 8: write 8 before querying, call F-3c18ad612b20 after the query, and return 1. F-22314fdee3bd is not called.

5. **Unchanged request exceeds the first quantity.**  
   Captured value and request both 12, first quantity 10: no explicit write, no F-3c18ad612b20 call, and return 1.

6. **Second quantity reaches equality.**  
   Request 5, first quantity 9, second quantity 5: F-22314fdee3bd occurs between queries. Return 1 without attached-object access, regardless of the changed decision.

7. **Persistent excess with null primary pointer.**  
   Request 5, quantities 9 and 8, captured attached-object pointer null: return 0 after F-22314fdee3bd and both queries. Earlier effects remain.

8. **Exact primary acceptance.**  
   Both quantities exceed the request and F-U03 returns exactly 1: return 1 without either fallback field read or F-U04.

9. **Unnormalized fallback result.**  
   F-U03 returns 2; fallback resolution and receiver are valid; F-U04 returns `0x80000000`: return unsigned 2,147,483,648.

10. **Stable replacement before fallback.**  
    F-U03 changes offset `0x48` from one valid receiver to another and returns zero. With no further relevant changes, both fallback reads observe the replacement. F-U04 is resolved through and receives that replacement, not the primary receiver.

11. **Distinct fallback observations.**  
    If the two fallback field reads can validly observe different pointers under the applicable execution model, this function does not require them to match. The target is resolved using the target-resolution observation, while the receiver comes from the receiver-argument observation. No relative order between those observations is promised.

12. **Unsigned comparison boundary.**  
    Request `0x80000000`, first quantity `0x7FFFFFFF`: the request is greater. F-22314fdee3bd is not called. F-3c18ad612b20 is called only if the changed decision is true.

13. **Stored-field mutation after capture.**  
    The captured value equals the request, but F-U02 changes the field before returning. The function still treats the value as unchanged and does not perform its conditional write.

## Uncertainties

The following remain **documented_only**:

- Semantic meanings of the gate, stored value, queried quantity, and action dependencies.
- Synchronization, thread-safety, reentrancy, and visibility guarantees.
- Permitted concurrent mutation and the execution model governing such mutation.
- Ownership, retention, release, allocation, and lifetime effects of dependencies.
- Concrete indirect-call implementations and receiver requirements.
- Meaning of unrestricted fallback results.
- Outcomes of invalid access, invalid dispatch, or abnormal dependency termination.
- Byte order and portability beyond the visible 32-bit representation.

The fallback expression establishes separate source-level field reads, not their relative order or any particular machine-instruction sequence. It supplies no stability guarantee and no fresh null checks.

No arithmetic transforms the requested value. Address formation uses fixed offsets. Validity of those calculations and subsequent accesses depends on the target address model and supplied storage; no portable overflow or wraparound behavior is promised.

“Limit adjustment,” “synchronization,” and “acceptance” remain intent-oriented descriptions rather than additional guarantees.

## Evidence

This contract derives from static inspection of the supplied function body and dependency map, not from the prior draft’s assertions.

Established evidence includes the entry-byte gate; bracketed stored-value read and conditional write; zero-request operation; unsigned quantity comparisons; branch-specific actions; captured and checked primary receiver; exact comparison of the primary result with 1; and propagation of the fallback result.

For fallback, the supplied expression independently obtains the attached-object field for target resolution and for the receiver argument. It provides no captured replacement pointer, no ordering guarantee between those evaluations, and no new null check for either read.

No runtime tests have been performed. Acceptance examples are conditional consequences of the visible behavior, not observed executions.
