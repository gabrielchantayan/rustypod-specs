<!-- Trusted provenance: F-69bdcb8fe46e; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-69bdcb8fe46e

## Purpose

F-69bdcb8fe46e performs a guarded, descriptor-selected initialization of a caller-supplied context. It checks initial eligibility, obtains two subordinate-object values, and invokes a preparation dependency. Successful preparation leads to temporary-value processing and a comparison.

Comparison rejection selects status 12 and invokes a context-level cleanup dependency. Comparison acceptance publishes a data address, writes a related address into the first subordinate object, and invokes two further dependencies.

A significant distinction is that the function reads the external table-base global twice on eligible paths. The first value is retained for later sizing. The second value is used only for initial table validation. These values are not established to be equal.

“Guarded,” “initialization,” “object,” “comparison,” and “cleanup” describe apparent roles. They do not establish synchronization, ownership, resource-release behavior, or application-level semantics.

## Interface

### Inputs and return value

The function receives two 32-bit values used as addresses:

- **Context address:** identifies storage containing the fields listed below.
- **Descriptor address:** is stored in the context and directly used for the initial selector read.

The supplied representation uses 32-bit integer values for addresses, fields, and the return status. A wider-pointer interface is not established.

The returned status is selected as follows:

| Condition | Result |
|---|---:|
| Initial selector is not unsigned 0 or 1 | 9 |
| Initial table validation fails | 9 |
| Either subordinate-object-producing call returns zero | 9 |
| Preparation returns nonzero | That exact 32-bit status |
| Preparation returns zero and comparison returns zero | 12 |
| Preparation returns zero and comparison returns nonzero | 0 |

These results apply only when all reached operations complete normally. The function does not visibly convert invalid memory accesses or abnormal dependency termination into statuses.

### Observed layouts

Offsets are byte offsets.

| Storage | Offset | Width or extent | Observed use |
|---|---:|---:|---|
| Context | `0x50` | 4 bytes | Stored descriptor address |
| Context | `0x54` | 4 bytes | Published data address |
| Context | `0x58` | Unknown extent | Address supplied to entry and exit dependencies |
| Context | `0x64` | 4 bytes | First subordinate-object value |
| Context | `0x68` | 4 bytes | Second subordinate-object value |
| Descriptor | `0x10` | 4 bytes | Selector |
| Table entry | `0` | 4 bytes | Initial validation value |
| Table entry | `4` | 4 bytes | Quantity used in later sizing |
| First subordinate object | `0x2c` | 4 bytes | Calculated related address |

Table entries are selected using a 12-byte stride. No meaning is established for the remaining bytes of an entry.

Two distinct local temporary regions each contain 40 bytes. Their internal formats are unknown. Their addresses are supplied to dependencies; no ownership transfer is established.

The extent of the context’s region beginning at `0x58` is unknown. Its proximity to other fields does not establish disjointness or overlap.

### Preconditions for normal completion

Every address actually read, written, or supplied to a dependency must satisfy the requirements of that access or dependency. The function does not visibly validate the context, descriptor, table storage, extracted data address, or subordinate-object storage.

A nonzero subordinate-object result is tested only against zero; it is not independently checked for sufficient size, alignment, or writability. Later accesses also depend on values reread after dependency calls, not merely on the original construction results.

## Behavior

### Entry and initial table selection

The first visible operation is F-338d16f9ce72, receiving the context address plus `0x58`. Its result does not control a branch.

The function then stores the supplied descriptor address at context offset `0x50`.

The initial reads occur in this order:

1. Read the external table-base global and retain that value for later sizing.
2. Read the four-byte selector at offset `0x10` from the supplied descriptor.
3. Test the selector as unsigned against 2.
4. Only if the selector is below 2, read the external table-base global again and use this second value to locate the validation entry.
5. Read that entry’s four-byte validation value and require it to equal the selector.

Thus, selector 0 or 1 can proceed; every other 32-bit selector selects status 9 without the second table-base read or a validation-entry read.

The validation address is the **second table base plus 12 times the initial selector**. Both later sizing expressions instead use the **first captured table base**.

There is no intervening dependency call between these two global reads. That does not establish a single snapshot, equality of the values, or synchronization against another writer. If the global changes between the reads, validation can use one table while later sizing uses another. Even if both reads yield the same address, the table contents are not established to remain unchanged.

Validation failure selects status 9 and skips both construction sequences. The descriptor-address store and first table-base read have already occurred.

### Subordinate-object construction

After successful validation, the function calls F-8157a180d901 with 56, then calls F-f13320f8b835. The latter’s result is stored at context offset `0x64`.

The visible representation does not explicitly pass the result of the size-request call into the object-producing call. Allocation, construction, and calling-convention relationships remain unknown.

If the first object-producing result is zero, that zero has already been stored. The second construction sequence is skipped and status 9 is selected.

Otherwise, the function calls F-8157a180d901 with 36, then calls F-bac40d036c97. The latter’s result is stored at context offset `0x68`.

A zero second result selects status 9. No explicit call to F-389a28346612 occurs on either construction-failure path. The function does not directly roll back the earlier field stores.

### Preparation

When both object-producing calls returned nonzero, F-d597506c21b6 receives the context address.

A nonzero preparation result is returned unchanged after the exit dependency. Temporary processing is skipped, and no explicit context cleanup call occurs.

A zero preparation result enables temporary processing. Zero remains the selected status unless comparison rejection replaces it with 12.

### Temporary processing and comparison

The function invokes the following dependencies in order:

- F-2057c4c921ad with the first 40-byte temporary region and the context address.
- F-19de4dea6bf0 with the second 40-byte temporary region.
- F-9ea93452e183 with the first temporary region and the four-byte value returned by F-19de4dea6bf0.
- F-93092e17776d with the second temporary region.

The comparison result is interpreted only as zero or nonzero. The second temporary’s finalization-like call occurs before either comparison branch is acted upon.

For a zero comparison result, the function selects status 12 and calls F-389a28346612 with the context address. It then calls F-93092e17776d with the first temporary region.

For a nonzero comparison result, the function performs the publication and initialization operations below, then calls F-93092e17776d with the first temporary region.

These calls do not establish whether either temporary owns memory, retains references, or requires resource release. “Finalization” describes call placement, not a proven dependency contract.

### Publication and related-address calculation

On comparison acceptance, F-28a4a5e2039e receives the first temporary region. Its four-byte result is stored at context offset `0x54`.

The function next calculates and stores a related address using:

- The result just returned by F-28a4a5e2039e.
- The **first captured table base**.
- A descriptor address reread from context offset `0x50`.
- A selector read from offset `0x10` of that descriptor.
- The four-byte quantity at offset `4` of the selected 12-byte table entry.

The related address is the extracted result plus 12 times this quantity. It is written to offset `0x2c` of the object whose address is read from context offset `0x64`.

This calculation does not reread the table-base global. A subsequent change to that global does not replace the retained base. Changes to memory reachable through the retained base can nevertheless affect the quantity read.

The later descriptor and selector are not required by a visible check to equal their initial values. No repeated unsigned eligibility check or table validation is performed. A dependency that changes context offset `0x50`, the referenced descriptor, or the relevant table contents can therefore change the calculation.

The relative evaluation order of independent reads within this expression is not established. The contract does establish that publication at `0x54` precedes this related-address store.

### Fill-like call and final object operation

After the related-address store, the function calls F-8ff0260f6859 with:

1. The four-byte value reread from context offset `0x54`.
2. A quantity obtained through another descriptor-and-selector read, using the **first captured table base**, multiplied by 12.
3. The value 255.

The descriptor address, selector, and table quantity are read again for this call. They are not specified as a cached copy of the preceding sizing calculation.

No dependency call separates the related-address store from preparation of these arguments. That does not make the reads interchangeable: the related-address write may alias storage subsequently read, and external mutation is not ruled out.

The argument pattern suggests filling a region with byte value `0xff`. The actual write behavior, count interpretation, and treatment of zero or negative counts belong to the unknown contract of F-8ff0260f6859.

After that dependency returns, F-f2dc8a5f0bbd receives the first-object address freshly read from context offset `0x64` and the first temporary region. F-93092e17776d then receives the first temporary region.

### Exit

Every represented normal-return path calls F-1333331b2da1 with the context address plus `0x58`, then returns the selected status.

The exit dependency’s result does not select the function’s status. Its effects are otherwise unknown.

The entry and exit calls suggest a protected region, but successful acquisition, mutual exclusion, and release semantics are not established.

## Side effects

### Direct writes and partial progress

When their paths are reached, direct writes occur in this order:

1. Supplied descriptor address to context offset `0x50`.
2. First object-producing result to context offset `0x64`.
3. Second object-producing result to context offset `0x68`.
4. Extracted data value to context offset `0x54`.
5. Calculated related address to the current first object’s offset `0x2c`.

The function is not visibly transactional:

- Eligibility failure does not directly clear existing object fields.
- First construction failure does not directly clear `0x68`.
- Construction or preparation failure does not directly clear `0x54`.
- Comparison rejection does not directly publish a new value at `0x54`.
- F-389a28346612 is called only on comparison rejection.

These are statements about this function’s direct operations. Dependencies may change the same fields, other memory, globals, or ownership state. No dependency is assumed to preserve them.

The function directly reads, but does not directly write, the external table-base global. Its first captured value is retained independently of the second validation read and any subsequent global changes.

### Arithmetic and aliasing

The initial selector and validation comparison are unsigned 32-bit operations. Consequently, `0xffffffff` fails the initial selector range test.

Later selector and quantity reads appear as signed 32-bit values. The table-entry address uses a stride of 12; the sizing operation separately multiplies the entry quantity by 12. Neither operation has a visible range or overflow check.

For representable, valid values, the related address is the extracted value plus the scaled quantity. Exact overflow behavior of the original execution environment is not established. The contract does not promise safe wrapping, rejection, or valid accesses for negative quantities, unchecked later selectors, or unrepresentable intermediate results.

Aliasing is not excluded. For example, storing the descriptor address can affect the initial selector read if the corresponding storage overlaps. The related-address store can affect the later fill-like arguments if its destination overlaps the context, descriptor, or table. Stable-memory examples require explicit noninterference assumptions.

## Dependencies

All roles below are descriptive. None establishes allocation, retention, release, synchronization, or preservation guarantees.

| Opaque ID | Observed interaction |
|---|---|
| F-338d16f9ce72 | Entry call with context plus `0x58` |
| F-1333331b2da1 | Exit call with context plus `0x58` on every represented normal return |
| F-8157a180d901 | Receives 56 before the first object-producing call and 36 before the second |
| F-f13320f8b835 | Produces the value stored at `0x64` |
| F-bac40d036c97 | Produces the value stored at `0x68` |
| F-d597506c21b6 | Receives the context; nonzero status is propagated |
| F-2057c4c921ad | Receives the first temporary and context |
| F-19de4dea6bf0 | Receives the second temporary and returns a four-byte comparison argument |
| F-9ea93452e183 | Receives the first temporary and that argument; zero rejects |
| F-93092e17776d | Receives each temporary at its specified finalization point |
| F-389a28346612 | Receives the context only on comparison rejection |
| F-28a4a5e2039e | Receives the first temporary and returns the published value |
| F-8ff0260f6859 | Receives the reread published value, scaled quantity, and 255 |
| F-f2dc8a5f0bbd | Receives the reread first-object address and first temporary |

## Acceptance examples

These are conditional consequences of static evidence, not executed tests. All assume valid reached accesses and normally returning dependencies.

1. **Ineligible selector.** With initial selector 2, the descriptor address is stored and the first table-base read occurs. The second table-base read, validation-entry read, and construction sequences are skipped. The exit dependency runs and the result is 9.

2. **Unsigned boundary.** Initial selector `0xffffffff` is rejected as unsigned, regardless of its possible signed interpretation as −1. It does not select an entry for initial validation.

3. **Validation mismatch.** With initial selector 1, the validation value is read from the second table base plus 12. A value other than 1 selects status 9 without construction.

4. **Different table-base reads.** Suppose the first global read yields table A, the global changes before the eligible path’s second read, and that second read yields table B. Validation uses table B. If it succeeds and execution reaches sizing, both sizing expressions use table A. If stable quantities at the selected entries are 5 in A and 8 in B, the sizing quantity is 5, producing a scale of 60—not 96. The validation field must match in B; no validation-field check is performed in A.

5. **First construction failure.** After successful eligibility, a zero result from F-f13320f8b835 is stored at `0x64`. The second construction sequence is skipped. Status 9 is returned without an explicit context cleanup call.

6. **Second construction failure.** A nonzero first result and zero second result are both stored. Status 9 is returned without F-389a28346612. No release of the first object is established.

7. **Preparation failure.** Any nonzero preparation status, including a negative 32-bit status, is propagated unchanged. Temporary processing is skipped.

8. **Comparison rejection.** After zero preparation and zero comparison results, the second temporary is finalized, context cleanup is called, and the first temporary is finalized. The result is 12; no direct publication to `0x54` occurs.

9. **Accepted stable sizing.** With nonzero comparison, extracted address P, stable quantity 5 in the table selected through the first captured base, and representable valid arithmetic, the related field receives P plus 60. If intervening writes do not change the reread inputs, F-8ff0260f6859 receives P, 60, and 255. The final object operation and first-temporary finalization follow; the result is zero.

10. **Zero quantity.** With stable quantity zero, the related address equals the extracted value. The fill-like dependency is still called with count zero. Whether it accepts a particular address with that count is unknown.

11. **Later mutation.** If an earlier dependency replaces the descriptor stored at `0x50`, sizing follows the replacement descriptor without renewed eligibility validation. If that dependency also changes the table-base global, sizing still uses the first captured base, not the replacement global value.

## Uncertainties

The following remain **documented_only**:

- Synchronization semantics of the entry and exit dependencies.
- Whether external mutation can occur between the two initial global reads; no single-snapshot guarantee is established.
- The allocation protocol connecting size requests with object-producing calls.
- Ownership and lifetime of subordinate objects, temporary contents, and the published data address.
- Cleanup effects, including whether objects are released or context fields reset.
- The semantic meaning, valid range, and units of the table quantity.
- Actual effects and error behavior of the fill-like and final object dependencies.
- Overflow, alignment, and address-validity rules beyond the observed widths.
- Relative evaluation order of independent reads within compound expressions or call arguments.
- Dependency behavior on faults, exceptions, allocation failure, or nonlocal exits.
- Caller restrictions on aliasing or concurrent mutation.

These unknowns do not authorize added validation, rollback, ownership transfer, or stability guarantees.

## Evidence

Established behavior comes solely from the supplied static function representation: field widths and offsets, unsigned initial eligibility, two distinct initial table-base reads, reuse of the first captured base for sizing, branch conditions, call arguments and order, direct writes, repeated reads, and status selection.

Inferred intent is limited to descriptive roles such as guarded initialization, object construction, temporary finalization, comparison, and memory filling. Those labels do not replace dependency contracts.

No runtime tests have been performed. Acceptance examples state conditional static behavior, not observed execution.
