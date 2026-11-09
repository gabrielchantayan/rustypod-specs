<!-- Trusted provenance: F-1151dcd9bead; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-1151dcd9bead

## Purpose

F-1151dcd9bead conditionally creates and registers an associated object for a subject, chooses its position within an existing collection, and configures its geometry and a binary property.

The established entry condition is a zero 32-bit field at subject offset `0x3C`. A nonzero value causes an immediate return without any dependency calls or explicit writes.

The apparent intent is to lazily create a presentation-related object and place it according to category and ordering information. That interpretation is inferred from the operations; the supplied evidence does not establish application-level names for the objects, ordering keys, or binary property.

## Interface

### Inputs and result

The function takes four inputs:

| Input | Established representation | Observed use |
|---|---|---|
| Subject | 32-bit address-bearing value | Base address for fields, argument to dependencies, and source of embedded geometry |
| Auxiliary input | Opaque 32-bit value | Forwarded only when the selector is zero |
| Selector | 32-bit value | Tested for zero; forwarded only when nonzero |
| Additional input | Opaque 32-bit value | Forwarded only when the selector is zero |

There is no explicit return value. Observable results consist of dependency calls, collection changes delegated to those calls, and an explicit write to the subject.

A negative selector follows the nonzero branch. Neither auxiliary input is inspected by this function.

### Directly accessed layouts

All listed scalar accesses are 32 bits.

| Base | Offset | Established use |
|---|---:|---|
| Subject | `0x34` | Category or lookup value |
| Subject | `0x3C` | Existing association check; later receives the constructed result |
| Subject | `0x80` | Geometry component; also the beginning of an address passed to two dependencies |
| Subject | `0x84` | Second geometry component |
| Related context | `0x30` | Subtrahend for the subject component at `0x84` |
| Related context | `0x34` | Subtrahend for the subject component at `0x80` |
| Related context | `0xF4` | Opaque constructor argument |
| Enumerated collection entry | `0x78` | Address of a candidate subject; zero means skip that entry |
| Candidate subject | `0x34` | Category compared with the current subject’s category |
| Lookup result | `0x00` | Address of a dispatch table |
| Dispatch table | `0x4C` | Indirect operation used to obtain a signed ordering result |

“Related context” means the result of F-ffe0d92c57b7. These role names do not establish concrete types or ownership.

### Preconditions and storage lifetime

On the nonzero-association path, only the subject field at `0x3C` must be readable for the visible operation to complete.

On the creation path, the function relies on valid addresses, compatible object layouts, valid dispatch tables, and successful dependency operations. It provides no explicit validation of those conditions.

Temporary storage includes:

- An eight-byte enumeration state.
- A two-component, eight-byte geometry value.
- Address-bearing cells passed to the indirect ordering operation.
- A cell into which the iterator writes the current collection entry.

Some dependency arguments point into this temporary storage. The evidence does not show whether callees retain those addresses. Correct completion requires either synchronous consumption or some other lifetime guarantee outside this function.

## Behavior

### 1. Existing-association gate

The subject’s 32-bit field at `0x3C` is read first.

- If nonzero, execution returns immediately.
- If zero, creation proceeds.

There is no refresh, reordering, geometry update, or finalization on the early-return path. The gate tests only for zero; it does not validate a nonzero association.

### 2. Context lookup and construction

The function calls F-ffe0d92c57b7 with the subject and retains its result as the related context.

It then chooses between two construction paths.

**Zero selector:**

1. Call F-8157a180d901 with size `0x130`, or 304 bytes.
2. Read the 32-bit value at related-context offset `0xF4`.
3. Call F-96a2bd5d6b0a with the allocation result, that context value, the subject, the auxiliary input, and the additional input.
4. Retain the constructor’s return value as the new association.

**Nonzero selector:**

1. Call F-8157a180d901 with the same 304-byte size.
2. Read the same related-context field.
3. Call F-c656f96f1369 with the allocation result, that context value, the selector, the subject, and the constant `1`.
4. Retain this constructor’s return value as the new association.

The association is the constructor return value, not necessarily the allocation result. No visible check verifies either result before execution continues.

### 3. Obtain the subject’s ordering results

A temporary address-bearing cell is initialized with the subject.

The function reads the subject field at `0x34` and passes it to F-583dc5fa4d66. Using the returned object’s dispatch table, it invokes the operation at byte offset `0x4C`, passing:

- The lookup object as receiver.
- The address of the temporary cell containing the subject.

The returned value is retained for signed comparisons with candidates in the same category.

Next, F-4ee56a7b8bff is called with the subject. Its result is retained as a 32-bit unsigned value for comparisons involving different categories.

These two results remain the reference values throughout the scan. The function does not recompute them after later dependency calls.

### 4. Initialize enumeration

The function calls F-9454f48a14c2 with the related context, then passes its result and the eight-byte enumeration storage to F-8ca21d3a8cca.

The remembered eligible collection entry initially has the zero value.

Repeated calls to F-b38a3aba0e18 receive the enumeration state and the address of the current-entry cell:

- A zero return indicates exhaustion.
- Any nonzero return causes the supplied entry to be examined.

No narrower interpretation of a nonzero return is established.

### 5. Choose a collection position

For each yielded entry, the function reads the candidate address at entry offset `0x78`.

If that address is zero, the entry is ignored. It neither triggers insertion nor replaces the remembered eligible entry.

For a nonzero candidate, the candidate field at `0x34` is compared with a fresh read of the subject field at `0x34`.

#### Matching categories

For equal category fields:

1. A temporary cell is assigned the candidate address.
2. F-583dc5fa4d66 is called with a fresh read of the subject’s category field.
3. The dispatch-table operation at offset `0x4C` is invoked with the lookup result and the address of the candidate cell.
4. The current collection entry becomes the remembered eligible entry.
5. The saved subject ordering result is compared with the returned candidate result as signed 32-bit integers.

If the subject result is strictly less than the candidate result, scanning stops and F-038dc53bb95a is called with the related context, new association, and current entry.

If the subject result is equal to or greater than the candidate result, scanning continues.

The ordering operation receives writable temporary storage. Its return value determines the comparison; no later decision shown here uses any replacement address it may write into that storage.

#### Different categories

For unequal category fields, the function calls F-4ee56a7b8bff again. Unlike the earlier invocation, this call has no explicit argument in the supplied representation. Its true input contract is therefore unresolved.

The returned value is interpreted as unsigned 32-bit data. The current entry becomes the remembered eligible entry.

- If the returned value is less than or equal to the saved subject value, scanning continues.
- If it is greater, scanning stops and F-038dc53bb95a is called with the related context, new association, and current entry.

A contract must not silently replace this argumentless invocation with an assumed candidate argument.

#### Exhaustion

If iteration ends without selecting a stopping entry:

- With no remembered eligible entry, F-24c97eeadaad receives the related context and new association.
- Otherwise, F-abfcbe19cf6a receives the related context, new association, and last remembered eligible entry.

Thus, entries with a zero candidate address do not become anchors, even when they are the last entries enumerated.

The dependency signatures suggest default insertion, insertion relative to a stopping entry, and insertion relative to a remembered entry. Exact “before” or “after” semantics are not established by this function alone.

### 6. Publish the association

After the selected collection-operation call returns, the function writes the constructor result to subject offset `0x3C`.

This publication occurs:

- After construction and collection placement.
- Before geometry configuration.
- Before binary-property configuration.
- Before the final subject update.
- Before enumeration finalization.

Dependencies invoked before publication are not guaranteed to observe the new association through the subject field.

### 7. Configure geometry

The function prepares two 32-bit differences in contiguous temporary storage:

| Temporary component | Difference |
|---|---|
| First | Subject at `0x84` minus related context at `0x30` |
| Second | Subject at `0x80` minus related context at `0x34` |

The first assignment precedes the second. The evidence does not establish the relative read order of the two operands within either subtraction.

The arithmetic is represented as 32-bit integer subtraction without visible range checking, saturation, or wider intermediates. Exact overflow semantics are not independently established.

After preparing both components, the function:

1. Reads the subject association field and calls F-e8e4e1462829 with it and the address at subject offset `0x80`.
2. Reads the association field again and calls F-a19ace379809 with it and the address of the temporary two-component value.

The second call uses the previously calculated differences. If the first call changes the subject or context geometry, those changes do not cause recomputation of the temporary value.

Conversely, the association argument is reloaded. If the first call changes subject offset `0x3C`, the second call receives the changed field value.

### 8. Configure a binary property and finish

F-3fac671648b8 is called with the subject. Its result is treated as unsigned 32-bit data and converted to a binary argument:

- Result `0`: argument `1`.
- Result `1`: argument `0`.
- Result greater than `1`: argument `0`.

This includes high-bit-set values: they produce zero, not a negative-value exception or alternate branch.

The function then:

1. Reloads subject offset `0x3C` and calls F-df6da5bd408b with that association and the binary argument.
2. Calls F-6d16652f2d03 with the subject and the address at subject offset `0x80`.
3. Calls F-d0d3f100effc with the enumeration state.
4. Returns.

No visible alternate cleanup path handles a dependency that fails to return normally.

## Side effects

### Established direct effects

The sole explicit write into caller-reachable object storage is publication of the constructor result at subject offset `0x3C`.

The function also writes its temporary cells and enumeration-related storage. It passes writable addresses to dependencies, including an address into the subject itself.

### Delegated effects

Dependency calls may allocate memory, construct objects, mutate a collection, configure the association, change subject geometry, or finalize enumeration state. Only the calls and their order are established here; the full effects of their implementations are unknown.

No dependency may be assumed to preserve globals, subject fields, or other reachable memory. In particular:

- Category fields are read repeatedly rather than universally cached.
- Geometry is read after collection placement.
- The association is reloaded for successive configuration calls.
- Changes made by dependencies can therefore affect later operations.

### Ownership and errors

There is no explicit deallocation of the 304-byte allocation or constructor result in this function. Whether construction or collection registration transfers ownership is documented_only.

There is no explicit status return, null-result handling, retry, rollback, or restoration of prior state. A failure after publication may leave the association visible with only some configuration completed.

## Dependencies

| Opaque ID | Observed role |
|---|---|
| F-ffe0d92c57b7 | Obtain related context from the subject |
| F-8157a180d901 | Request 304 bytes of storage |
| F-96a2bd5d6b0a | Construct the association for a zero selector |
| F-c656f96f1369 | Construct the association for a nonzero selector |
| F-583dc5fa4d66 | Obtain an object used for indirect signed ordering |
| F-4ee56a7b8bff | Produce unsigned ordering data; later invocation has unresolved inputs |
| F-9454f48a14c2 | Obtain enumeration input from the related context |
| F-8ca21d3a8cca | Initialize enumeration storage |
| F-b38a3aba0e18 | Advance enumeration and supply an entry |
| F-038dc53bb95a | Place the association relative to a stopping entry |
| F-24c97eeadaad | Place the association when no eligible entry was remembered |
| F-abfcbe19cf6a | Place the association relative to the last eligible entry |
| F-e8e4e1462829 | Receive the association and subject geometry address |
| F-a19ace379809 | Receive the association and calculated geometry differences |
| F-3fac671648b8 | Produce the value converted to a binary argument |
| F-df6da5bd408b | Receive the association and binary argument |
| F-6d16652f2d03 | Perform the final subject-and-geometry operation |
| F-d0d3f100effc | Finalize enumeration state |

The indirect dispatch operation has no supplied opaque function ID. Its established interface is a lookup receiver plus a pointer to an address-bearing cell, returning a signed 32-bit comparison value. Its additional effects remain unknown.

## Acceptance examples

These are conditional acceptance criteria, not executed tests. Unless stated otherwise, dependencies return normally and do not introduce unrelated mutations.

1. **Existing association.** Given a nonzero subject field at `0x3C`, the function returns without invoking any dependency or changing that field.

2. **Zero selector.** Given an initially zero association and selector zero, the allocation request is 304 bytes. F-96a2bd5d6b0a receives the auxiliary and additional inputs. F-c656f96f1369 is not called.

3. **Nonzero selector.** Given selector `−1` represented in 32 bits, the nonzero constructor path is selected. F-c656f96f1369 receives that selector and constant `1`; the auxiliary and additional inputs are not forwarded.

4. **No eligible entries.** If iteration immediately exhausts, or every yielded entry has a zero field at `0x78`, placement uses F-24c97eeadaad.

5. **Strict signed stopping condition.** For a matching-category candidate, subject result `−2` and candidate result `1` trigger F-038dc53bb95a at that entry. Equal results do not trigger stopping.

6. **Unsigned boundary.** For a different-category candidate, a saved subject value of `0xFFFFFFFF` and a later result of `0` continue scanning. Signed comparison would give a different answer and is not acceptable.

7. **Skipped trailing entry.** If one eligible entry is followed only by entries with zero candidate addresses, exhaustion passes that eligible entry to F-abfcbe19cf6a, not a trailing skipped entry.

8. **Geometry layout.** Given subject components `0x80 = 40`, `0x84 = 90` and context components `0x30 = 20`, `0x34 = 5`, F-a19ace379809 receives temporary components `70`, then `35`.

9. **Association reload.** If F-e8e4e1462829 changes subject offset `0x3C`, F-a19ace379809 receives the replacement field value. Its geometry argument still contains the differences calculated before that change.

10. **Binary conversion.** Results `0`, `1`, `2`, and `0xFFFFFFFF` from F-3fac671648b8 produce respective binary arguments `1`, `0`, `0`, and `0`.

## Uncertainties

The following are documented_only rather than assumed:

- Exact object types and semantic names of category, geometry, and ordering fields.
- The actual input convention of the later F-4ee56a7b8bff invocation.
- The identity and complete contract of the indirect dispatch operation.
- Exact collection-placement direction for the three placement dependencies.
- Allocation failure behavior, constructor failure behavior, and ownership transfer.
- Whether dependencies retain temporary addresses or mutate shared state.
- Overflow behavior beyond the visible 32-bit arithmetic.
- Failure unwinding, concurrency safety, and reentrant behavior.

Lazy association creation and ordered collection placement are inferred intent. Thread-safe one-time initialization is not established: the visible gate contains no synchronization.

## Evidence

This contract is based solely on static inspection of the supplied function representation and opaque dependency mapping. Established observations include field offsets, scalar widths, call arguments visible in that representation, branch conditions, comparison signedness, explicit publication timing, and subsequent call order.

Dependency names do not establish their implementations. No implementation access, runtime execution, or tests were used. Acceptance examples describe required behavior conditional on the stated dependency outcomes; they are not reports of observed execution.
