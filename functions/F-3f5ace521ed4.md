<!-- Trusted provenance: F-3f5ace521ed4; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-3f5ace521ed4

## Purpose

F-3f5ace521ed4 conditionally replaces a selected four-byte field with a value from a separate global source.

The selector determines a fixed region or a region derived from a global pointer. The function reads the selected field at byte offset 4 and tests bit 17. If that bit is set, it returns without an explicit store. Otherwise, it repeats the region-selection process and replaces the entire destination field.

Region selection includes repeated reads of the global pointer. These reads are part of the supplied static representation even when their resulting addresses are subsequently discarded. They must not be collapsed into a single read in this contract.

A configuration or initialization purpose is plausible but unestablished. No device-specific meaning is assigned to the regions, the tested bit, or the replacement value.

## Interface

### Input

The function accepts one integer selector. It compares that selector with 0, 1, and 2; it performs no arithmetic on the selector and exposes no invalid-selector response.

The behavioral model uses the represented signed 32-bit integer interpretation. Independent confirmation of the machine-level argument width, signedness, and calling convention is unavailable.

| Selector | Region ultimately selected |
|---|---|
| 0 | A fixed region |
| 1 | The global pointer value obtained by the first and only pointer read in that selection |
| 2 | The global pointer value obtained by the third pointer read in that selection, plus 16,384 bytes |
| Any other value | The global pointer value obtained by the second pointer read in that selection, plus 32,768 bytes |

Under the signed 32-bit interpretation, negative selectors and positive selectors greater than 2 use the last row. Selection is categorical, not a selector multiplied by a stride.

### External memory resources

The function uses three externally supplied resources:

- **Fixed region:** selected when the selector is 0.
- **Global pointer storage:** contains the pointer read during every nonzero selection.
- **Replacement source:** supplies the four-byte value used on the replacement path.

Each ultimately selected region is accessed as follows:

| Relative byte range | Explicit field access |
|---|---|
| 0–3 | None |
| 4–7 | One 32-bit read during inspection, or one four-byte write during replacement |
| 8 onward | None |

The decision mask is `0x00020000`, corresponding to bit 17 with zero-based numbering. The inspected field is treated as an unsigned 32-bit value for that test.

The replacement is four bytes wide. The source value is not masked or combined with the inspected field.

The pointer width, region sizes, byte order, and physical properties of the storage are not established.

### Output and errors

The function returns no value. Its normal outcomes are:

- Return after the initial field inspection, without an explicit store.
- Perform one full-field replacement, then return.

It does not return a status, the inspected value, or the replacement value. No explicit error-reporting or access-failure recovery path appears.

### Preconditions for normal completion

All represented global-pointer reads must be permitted. The initially selected field must permit a four-byte read. If the inspected bit is clear, the replacement source must permit its read, and the destination selected by the second selection must permit a four-byte write.

The addressing operations must also be valid under the execution environment’s rules. This includes intermediate pointer-offset calculations whose results are later overwritten.

No null check, bounds check, alignment check, or overflow check is visible. The evidence does not establish what happens when a required access or address calculation is invalid.

## Behavior

### Region selection

The same selection process is used before inspection and, conditionally, before replacement.

**Selector 0:** select the fixed region. This selection performs no explicit read of the global pointer.

**Every nonzero selector:** first read the global pointer and initially select that value without an added region offset.

- For selector 1, selection ends with this first pointer value.
- For every other nonzero selector, read the global pointer again and calculate an address 32,768 bytes beyond this second value.
- For selector 2 specifically, subsequently read the global pointer a third time and calculate an address 16,384 bytes beyond this third value. This last address replaces the previously calculated address.

Consequently, selector 2 includes the intermediate 32,768-byte calculation even though its final offset is 16,384 bytes.

| Selector class | Pointer reads per selection | Address calculations in represented order | Read determining the final region |
|---|---:|---|---|
| 0 | 0 | Fixed region selection | None |
| 1 | 1 | First pointer value, no region offset | First |
| 2 | 3 | First pointer value; second value plus 32,768; third value plus 16,384 | Third |
| Other nonzero | 2 | First pointer value; second value plus 32,768 | Second |

Earlier selected addresses are overwritten. The function does not explicitly read a region field through those discarded addresses. Nevertheless, the preceding pointer reads and address calculations remain represented operations. Their lack of a final data contribution does not establish that they are side-effect-free.

### Initial inspection

After completing the first selection, the function reads four bytes at offset 4 from the final selected region and tests bit 17.

This is a bit-presence test. Any inspected value with bit 17 set takes the return path, regardless of its other bits.

On that path:

- No second region selection occurs.
- No explicit replacement-source read occurs.
- No explicit store occurs.

Any global-pointer reads required by the first selection have already occurred. In particular, selector 2 performs all three selection reads before inspecting the field, even when inspection leads immediately to return.

“No explicit store” does not mean “no external effects.” Reads may have effects in an environment with active or memory-mapped storage.

### Conditional replacement

If bit 17 of the inspected value is clear, the function performs the entire selection process again with the same selector.

The second selection does not reuse the pointer value that determined the inspected address. For nonzero selectors it performs a new sequence of one, two, or three global-pointer reads, as specified above.

After completing that selection, the function reads the replacement source and writes four bytes at offset 4 in the newly selected destination region. It then returns.

The write replaces the complete field:

- No original bits are intentionally preserved.
- The replacement is not restricted to bit 17.
- The replacement value may itself have bit 17 set or clear.
- There is no explicit read or bit test of the second selected field before writing.
- There is no post-write verification.

Thus, the inspected field and written field need not be the same location. Even within one selection, earlier pointer reads need not equal the final read that determines its address.

### Represented access order and counts

The represented order is:

1. Complete the first selection, including all required pointer reads and intermediate address calculations.
2. Read the final selected field at offset 4.
3. Test bit 17 and return if it is set.
4. Otherwise, complete a second selection, including its full pointer-read sequence.
5. Read the replacement source.
6. Store four bytes at offset 4 in the second selected region.

The explicit global-pointer read totals are:

| Selector class | Return after inspection | Replacement path |
|---|---:|---:|
| 0 | 0 | 0 |
| 1 | 1 | 2 |
| 2 | 3 | 6 |
| Other nonzero | 2 | 4 |

Every normally completed invocation has one explicit selected-field inspection. The replacement path additionally has one explicit replacement-source read and one explicit four-byte store.

These counts distinguish operations by their roles in the representation. They do not imply distinct physical addresses: the resources might alias.

This ordering describes the supplied static representation, not independently verified machine transactions. Emitted load counts, instruction scheduling, volatile behavior, transaction widths, barriers, and atomicity remain unknown.

### Address arithmetic and boundaries

All stated offsets are byte offsets. After a selection’s final pointer read, the effective field displacement is:

| Selector class | Field displacement from the determining pointer value |
|---|---:|
| 1 | 4 bytes |
| 2 | 16,388 bytes |
| Other nonzero | 32,772 bytes |

For selector 2, the intermediate address derived from the second pointer read has a 32,768-byte region offset, but no explicit field access is made through it.

There is no range progression beyond these cases. For example, selectors 3 and 2,147,483,647 select the same fallback class under the represented signed 32-bit model.

No modular-addressing guarantee follows from the offsets. Pointer overflow, wrapping, and invalid intermediate addresses remain subject to unknown execution-environment rules.

## Side effects

The function contains at most one explicit store per invocation. It writes four bytes to the field selected during the second selection.

No explicit assignment to global-pointer storage or replacement-source storage appears. However, the destination may overlap either resource; disjointness is not established. A destination store could therefore affect storage serving another role.

Repeated pointer reads also cannot be assumed inert. If such reads have external effects, those effects may occur even for pointer values later discarded. Similarly, field reads and replacement-source reads may have environment-specific effects.

The function does not explicitly allocate, release, retain, or transfer ownership of memory. It uses externally supplied storage without establishing its lifetime, exclusivity, or ownership.

No locking, retry, atomic read-modify-write operation, or synchronization protocol is visible. The interval between inspection and replacement permits state changes in any environment that allows them. The initial bit test is not a guarantee that the eventual destination remains eligible for replacement.

## Dependencies

No calls to other functions appear in the supplied representation. There are no additional opaque function dependency IDs to enumerate.

The external dependencies are:

- The fixed region.
- The global-pointer storage and the values returned by its successive reads.
- The replacement source.
- The execution environment’s address-calculation and memory-access rules.

No callee contract is available or needed to explain the visible control flow. Nothing establishes ownership, initialization, read stability, or fault recovery for the memory dependencies.

## Acceptance examples

These are conditional expectations derived from static evidence, not executed tests. Unless an example says otherwise, assume valid addressing, ordinary readable and writable storage, stable values, and no resource overlap.

### Fixed-region return

Given selector 0 and fixed-region field value `0x00020000`:

- The function reads the field at fixed-region offset 4.
- Bit 17 is set, so it returns.
- There are no explicit global-pointer reads, replacement-source reads, or stores.

The same return decision applies to `0xFFFFFFFF`.

### Selector 1 full-field replacement

Given selector 1, a stable global pointer, inspected value `0x00000001`, and replacement value `0xA5A55A5A`:

- The first selection reads the global pointer once.
- Inspection occurs four bytes beyond that pointer value.
- Bit 17 is clear.
- The second selection reads the global pointer once more.
- The replacement source is then read, and the entire destination field becomes `0xA5A55A5A`.

There are two explicit pointer reads in total. The original low bit is not separately preserved.

### Selector 2 return after three pointer reads

Given selector 2 and a stable global pointer:

- The first selection reads the pointer without an offset.
- It reads the pointer again and calculates the 32,768-byte-offset address.
- It reads the pointer a third time and calculates the final 16,384-byte-offset address.
- The field inspection occurs 16,388 bytes beyond the third pointer value.

If that field has bit 17 set, the function returns after these three pointer reads. It does not inspect a field through the intermediate 32,768-byte-offset address.

### Selector 2 replacement

Given selector 2, stable resources, and an inspected field with bit 17 clear:

- The inspection selection performs three pointer reads.
- The replacement selection performs another three pointer reads.
- The replacement-source read follows all six pointer reads.
- The store targets the final pointer value plus 16,388 bytes.

With a stable pointer, the inspected and written field addresses coincide. This does not eliminate any of the six represented pointer reads.

### Fallback selector

Given selector −1 or 3 under the signed 32-bit model:

- Each selection reads the global pointer twice.
- The first value is initially selected and then discarded.
- The second value plus 32,768 bytes determines the region.
- Its field is four bytes farther on.

An inspection-only invocation has two pointer reads. A replacement invocation has four. No invalid-selector response is represented.

### Pointer changes within and between selections

Suppose selector 2 is used. The first selection obtains pointer values identifying bases A, B, and C, in that order.

- Inspection uses C plus 16,388 bytes.
- No field is explicitly inspected through A or B.
- The intermediate address B plus 32,768 bytes is nevertheless calculated.

If inspection permits replacement and the next selection obtains bases D, E, and F:

- The destination is F plus 16,388 bytes.
- D and E do not determine the destination.
- The destination field is overwritten without a separate bit test, even if its bit 17 is set.

This example is conditional on an environment permitting different values across the represented reads. It does not assert a concurrency or machine-access guarantee.

### Replacement and subsequent calls

If replacement writes a value with bit 17 set, a later invocation inspecting that same unchanged field returns without replacement. If replacement leaves bit 17 clear, a later invocation may replace it again.

This is conditional sequential behavior, not an established one-time-initialization protocol.

## Uncertainties

The following remain **documented_only**:

- **Machine interface:** exact calling convention, argument representation, and pointer width.
- **Integer model:** independent confirmation of the represented 32-bit selector and unsigned-field interpretation.
- **Emitted accesses:** whether machine execution preserves each represented pointer read, its timing, and its transaction width.
- **Memory semantics:** endianness, alignment, volatility, read side effects, write side effects, atomicity, and ordering guarantees.
- **Address calculations:** behavior of invalid or overflowing pointer arithmetic, including discarded intermediate calculations.
- **Resource validity:** initialization, allocation sizes, lifetime, and accessibility.
- **Mutability:** whether successive pointer or source reads can observe changes.
- **Aliasing:** whether the fixed region, selected regions, pointer storage, and replacement source overlap.
- **Failures:** whether invalid operations trap, fault, abort, or behave otherwise.
- **Intent:** the meaning of bit 17 and the purpose of the replacement.

These unknowns do not authorize collapsing represented reads or adding validation, synchronization, ownership behavior, or error reporting.

## Evidence

**Established static evidence:** The supplied representation contains two copies of the same conditional region-selection process. Each selection performs zero global-pointer reads for selector 0, one for selector 1, three for selector 2, and two for other nonzero selectors. Every nonzero selection first reads the unoffset pointer. Non-1 selectors then read it again for a 32,768-byte offset; selector 2 subsequently reads it a third time for the final 16,384-byte offset.

The first selection precedes a four-byte field read at offset 4 and a bit-17 test. A set bit causes immediate return. Otherwise, the second complete selection precedes a replacement-source read and one four-byte field store. No function calls or explicit error-reporting paths appear.

**Inferred intent:** The tested bit may guard configuration or initialization. This interpretation supplies no required behavior and assigns no established meaning to the resources.

**Verification boundary:** This contract is based solely on the supplied static representation. No implementation access, runtime execution, or tests were used.
