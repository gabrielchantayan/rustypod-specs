<!-- Trusted provenance: F-65dd7de05cfd; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-65dd7de05cfd

## Purpose

F-65dd7de05cfd conditionally invokes preliminary processing, chooses between two position-related processing paths when a buffer-related field is nonzero, and performs a final dependency invocation followed by three bookkeeping resets on normal completion.

Established behavior consists of field reads and writes, arithmetic, ordered dependency calls, and three explicit failure conditions. The apparent intent is to reconcile buffered stream-like state. That interpretation is inferred; the supplied representation does not establish the object’s domain, the units of its fields, or the semantics of its dependencies.

In particular, calls to F-dd25b8fb230c establish only the invocation and the ordered pair of address arguments. They do **not** establish copying, transfer direction, or resulting storage contents.

The function returns a 32-bit value:

- **0:** execution reached normal completion and performed the final direct resets.
- **0xffffffff:** execution encountered one of the explicitly checked failure conditions.

Failure is not transactional. No rollback or compensating operation is visible.

## Interface

### Input and result

The sole input is a pointer to an object accessed as consecutive 32-bit words. The result is one 32-bit value.

Direct accesses extend through byte offset 71. Dependencies may require additional object storage, reachable buffers, or dispatch metadata. No null, size, alignment, buffer-bound, or arithmetic-range checks are visible.

The labels below describe usage at this call site, not established domain meanings.

| Byte offset | Width or form | Observed use |
|---|---|---|
| 0 | 32-bit value | Dispatch root for the preliminary indirect call. |
| 20 | 32-bit value | Buffer-base operand and dependency argument. |
| 24 | 32-bit value | Buffer-limit operand, processing gate, iterative target, and assignment destination. |
| 28 | 32-bit value | Assignment destination after successful buffer processing. |
| 32 | 32-bit value | Cleared on normal completion. |
| 36 | 32-bit value | Controls preliminary dispatch; cleared on normal completion. |
| 40 | 32-bit value | Cleared last on normal completion. |
| 44 | Address within object | Argument to F-74370bf369b4; subobject extent unknown. |
| 52 | 32-bit value and subobject address | Arithmetic destination, preparation argument, and argument to F-dd25b8fb230c. |
| 56 | 32-bit value | Receives the iterative state after the iterative dependency sequence. |
| 60 | Address within object | Argument to F-504c4fc15e87 and F-dd25b8fb230c; subobject extent unknown. |
| 64 | 32-bit value | Initial value of the iterative mutable state. |
| 68 | 32-bit value | Adjustment operand in buffer arithmetic. |

These locations need not be independent semantic fields. For example, a dependency receiving the address at offset 60 might access offset 64. Its actual access extent is not supplied.

### Preconditions

Defined operation requires:

- An object valid for all reached direct reads and writes.
- Valid dispatch metadata and a callable masked target when preliminary dispatch occurs.
- Arguments and reachable storage satisfying every reached dependency’s requirements.
- Arithmetic and address representations for which the visible operations are meaningful.
- Dependency behavior compatible with the provided temporary-storage extents.

Termination additionally requires the iterative dependency to report the checked failure status or eventually make progress equal the fixed target. The function does not enforce this requirement.

### Temporary storage and ownership

The invocation provides:

- One 512-byte scratch region.
- Two distinct 8-byte scratch regions.
- An adjacent pair of 32-bit words initialized before shared validation.
- Addressable 32-bit cells for iterative state, progress, and scratch-result output.

The scratch byte regions have no visible initial contents. The iterative cells and validation pair receive the initializations described below.

No explicit heap allocation or release appears in this function. This does not establish whether any dependency allocates, releases, retains pointers, changes globals, or changes ownership. Temporary addresses are passed to dependencies, but no retention or lifetime guarantee is supplied.

## Behavior

### 1. Preliminary dispatch

The first direct object read examines offset 36.

If it is zero, execution proceeds to the buffer-presence check without preliminary calls.

If it is nonzero:

1. Obtain the dispatch root from offset 0.
2. Read a 32-bit target value at byte displacement 40 from that root.
3. Clear the target value’s two least significant bits.
4. Invoke the resulting target with the object pointer and the 32-bit value `0xffffffff`.
5. Invoke F-078279ab0f90 with the object pointer first and the indirect result second.
6. Return `0xffffffff` if F-078279ab0f90 returns nonzero; otherwise continue.

The indirect result is not directly checked for failure. F-078279ab0f90 supplies the checked result.

Both calls precede the buffer-presence read. Their mutations may therefore affect subsequent processing. No assumption that they preserve object fields or global state is justified.

### 2. Buffer gate and branch selection

Read offset 24.

If it is zero, skip all position-related processing, shared validation, buffer operation, and pointer reassignment. Proceed to the final dependency invocation and resets.

If it is nonzero:

1. Retain that value as the buffer-limit snapshot.
2. Read and retain offset 20 as the buffer-base snapshot.
3. Read and retain offset 68 as the adjustment snapshot.
4. Invoke F-74370bf369b4 with the address at offset 44 and retain its 32-bit result as a processing token.
5. Invoke F-03b3b063bdbc and interpret its result as a signed 32-bit factor.

A factor less than 1 selects iterative processing. A factor of 1 or greater selects arithmetic processing.

The arithmetic branch uses the retained base, limit, and adjustment. The iterative branch reloads the relevant object fields after both calls. This distinction matters if either dependency mutates the object.

### 3. Arithmetic processing

For a positive factor, first invoke F-dd25b8fb230c with:

- First argument: address at offset 52.
- Second argument: address at offset 60.

No result is checked. No copy semantics or post-call contents are established.

After that call, read the 32-bit value at offset 52. Increase it by:

**factor × (((buffer-limit snapshot − buffer-base snapshot) shifted right by one) − adjustment snapshot)**

Write the result to offset 52.

The subtraction, shift, multiplication, and addition are represented using 32-bit signed integer operands. The shift occurs after the limit-minus-base subtraction and before the adjustment subtraction. It must not be replaced unconditionally with division by two: negative odd values can distinguish signed right-shift behavior from division that truncates toward zero.

There is no explicit overflow or underflow check. Exact overflowing results and target-dependent signed-shift behavior are not established by the supplied representation.

The value being incremented is read after F-dd25b8fb230c returns, while the three buffer operands come from the earlier snapshots. No direct assignment to offset 56 occurs on this branch, although dependency effects remain unknown.

### 4. Iterative processing

For a factor below 1:

1. Initialize the scratch-result cell to zero.
2. Initialize the progress cell to zero.
3. Read offset 64 into the mutable state cell.
4. Reload offset 24 as the fixed loop target.
5. Initialize an accumulated amount to zero.
6. Form the initial processing-start value from fresh reads as **buffer base + 2 × adjustment**.

While progress differs from the fixed target, invoke F-605d034d51cf with these arguments in order:

1. The retained processing token.
2. Address of the mutable state cell.
3. Current processing-start value.
4. Fixed target.
5. Address of the progress cell.
6. The 512-byte scratch region.
7. The first 8-byte scratch region.
8. Address of the scratch-result cell.

If the returned status is exactly **2**, return `0xffffffff` immediately. That iteration does not contribute to accumulation, and no later processing stage is reached.

For every other status:

- Add **scratch-result value − address of the 512-byte scratch region** to the accumulated amount.
- Use the current progress value as the next processing-start value.
- Recheck progress against the fixed target.

The dependency’s returned status is not the next processing-start value. Neither a different nonzero status nor a negative status independently causes failure here.

The mutable state, progress, scratch-result cell, and scratch regions persist across iterations. The caller does not reinitialize them between calls. Thus, if a dependency leaves a cell unchanged, its previous value remains available to subsequent caller operations.

There is no iteration bound, monotonic-progress check, or local validation that the scratch-result value identifies a position within the 512-byte region. Accumulation uses the visible 32-bit address/integer representation.

After the loop terminates:

1. Invoke F-504c4fc15e87 with the first 8-byte scratch region, the address at offset 60, and the accumulated amount.
2. Invoke F-dd25b8fb230c with the address at offset 52 first and the first 8-byte scratch region second.
3. Write the mutable state cell to offset 56.

Neither dependency result is checked. Their calls do not establish the contents produced at either supplied address.

A zero-iteration path is possible if dependencies change offset 24 to zero before its reload. Progress then initially equals the target. The two post-loop calls still occur, despite no iterative call having populated the scratch regions.

### 5. Shared preparation and validation

Both processing branches converge on this sequence:

1. Initialize the first word of the adjacent temporary pair to `0xffffffff`.
2. Initialize the second word to zero.
3. Invoke F-5eb783ea358d with the second 8-byte scratch region, the object pointer, the current value at offset 52, and zero.
4. Invoke F-ed5811f8a19e with that second scratch region and the address of the initialized two-word area.

If F-ed5811f8a19e returns nonzero, return `0xffffffff`.

This failure happens after preceding dependency effects and any direct position-related writes. The function does not undo them. It does not reach the buffer operation, the two buffer-pointer assignments, or the final dependency invocation and resets.

The initialized pair’s meaning is unknown. Its first word is an all-ones bit pattern; the call site alone does not establish a sentinel convention or comparison rule.

### 6. Buffer operation and pointer assignments

If validation returns zero, invoke F-5f0243819f69 with:

- First argument: the current buffer-base value at offset 20.
- Second argument: **current buffer limit − 2 × current adjustment**, using offsets 24 and 68.

These values are read after validation; they are not the snapshots used in arithmetic processing. Any dependency result is ignored.

After the call, reread offsets 20 and 68 and compute:

**buffer base + 2 × adjustment**

Write that same computed 32-bit value first to offset 24 and then to offset 28.

Consequently, changes made by F-5f0243819f69 to the base or adjustment can affect both assignments. The second assignment uses the already computed value, not a fresh computation after the first assignment.

The second argument to F-5f0243819f69 is established only as the stated expression. Its interpretation as an address, length, bound, or other quantity is unknown.

### 7. Final dependency invocation and resets

This stage follows successful nonempty-buffer processing or the initial zero buffer-limit case.

Perform these operations in order:

1. Invoke F-dd25b8fb230c with the address at offset 60 first and the address at offset 52 second.
2. Write zero to offset 36.
3. Write zero to offset 32.
4. Write zero to offset 40.
5. Return zero.

No dependency result is checked. The ordered address arguments are established; copying, transfer direction, and resulting subobject contents are not.

## Side effects

The direct object writes are:

| Condition | Direct writes in order |
|---|---|
| Arithmetic branch | Computed value to offset 52. |
| Completed iterative branch | Mutable state to offset 56, after its two post-loop calls. |
| Successful validation and buffer operation | Same computed value to offsets 24 and 28, in that order. |
| Normal completion | Zero to offsets 36, 32, and 40, in that order. |

Dependencies can have additional effects not bounded by this table. Receiving an object or scratch address does not establish the dependency’s exact read/write extent, ownership behavior, or global-state preservation.

All explicit error paths return without local cleanup or rollback:

- Preliminary failure follows the indirect call and F-078279ab0f90.
- Iterative status 2 follows that iteration’s dependency call.
- Validation failure follows position-related processing and both shared preparation calls.

A zero result guarantees that the final direct resets were reached. It does not prove a particular state transformation by any unchecked dependency.

## Dependencies

| Opaque ID | Established call-site role |
|---|---|
| F-078279ab0f90 | Receives the object and preliminary indirect result; nonzero causes failure. |
| F-74370bf369b4 | Receives the address at offset 44 and returns the iterative processing token. |
| F-03b3b063bdbc | Returns the signed factor selecting arithmetic or iterative processing. |
| F-605d034d51cf | Receives iterative state, progress, and scratch addresses; exactly 2 causes failure. |
| F-504c4fc15e87 | Receives the first 8-byte scratch region, address at offset 60, and accumulated amount. |
| F-dd25b8fb230c | Receives two ordered addresses at three possible stages; transformation and direction are unknown. |
| F-5eb783ea358d | Receives the second 8-byte scratch region, object, current offset-52 value, and zero. |
| F-ed5811f8a19e | Receives that temporary and the initialized word-pair address; nonzero causes failure. |
| F-5f0243819f69 | Receives two buffer-derived values after validation; no result is checked. |

The indirect target has no supplied opaque dependency ID. Its selection mechanism and two visible arguments are established; its behavior remains documented_only.

## Acceptance examples

These are conditional static consequences, not executed tests.

1. **No pending state and no buffer.** With offsets 36 and 24 both zero, no preliminary or buffer-processing calls occur. F-dd25b8fb230c receives the addresses at offsets 60 and 52, in that order. Offsets 36, 32, and 40 are then cleared, and the function returns zero. No particular subobject contents are implied by the call.

2. **Preliminary failure.** With nonzero offset 36, a nonzero F-078279ab0f90 result produces `0xffffffff` before the buffer-presence read. Effects of both preliminary calls may remain.

3. **Positive-factor calculation.** With retained base 1000, limit 1020, adjustment 2, and factor 3, the increment is 24. If offset 52 is 100 after the branch’s F-dd25b8fb230c invocation, the direct arithmetic write makes it 124. This assumes representable arithmetic.

4. **Snapshot distinction.** If the factor-selection dependencies change the object’s base or limit, the positive-factor increment still uses the earlier snapshots. The iterative branch instead uses its subsequent field reads.

5. **Iterative accumulation.** If two continuing iterations leave scratch-result values 6 and 4 bytes beyond the 512-byte region’s start, and the second reaches the fixed target, the accumulated argument to F-504c4fc15e87 is 10, assuming representable arithmetic.

6. **Exact iterative failure.** A status of 2 stops processing before that iteration’s accumulation. A status of 3 does not itself fail; if progress has reached the target, processing proceeds to the post-loop calls.

7. **Validation failure after mutation.** A nonzero F-ed5811f8a19e result returns `0xffffffff` without the buffer operation, pointer assignments, or final invocation and resets. Earlier writes are not rolled back.

8. **Post-call pointer calculation.** If F-5f0243819f69 leaves base 2000 and adjustment 3, the subsequent direct assignments write 2006 to offset 24 and then offset 28, irrespective of earlier snapshots.

9. **Zero iterative calls.** If the initial buffer check is nonzero but the reloaded iterative target is zero, no F-605d034d51cf call occurs. F-504c4fc15e87 still receives an accumulated amount of zero, followed by F-dd25b8fb230c and the offset-56 assignment. Whether the scratch-dependent calls are valid requires their unavailable contracts.

## Uncertainties

- **documented_only — F-dd25b8fb230c semantics:** Ordered address arguments do not establish copying, source/destination roles, transfer direction, or resulting contents.
- **documented_only — Arithmetic:** No checked arithmetic is visible. Exact overflow outcomes and negative signed-right-shift behavior require target semantics not supplied here.
- **documented_only — Units and representation:** Buffer operands appear address-like, but units, encoding, and the meaning of doubling the adjustment are unknown.
- **documented_only — Dependency extents:** Complete layouts and access sizes for the subobjects at offsets 44, 52, and 60 are unavailable.
- **documented_only — Scratch validity:** Scratch bounds, output initialization guarantees, valid address differences, and permitted accumulated amounts are unknown.
- **documented_only — Progress and termination:** No supplied dependency contract guarantees progress, target reachability, or termination.
- **documented_only — Ownership and global effects:** No dependency preservation, retention, acquisition, or release guarantee is established.
- **documented_only — Fine-grained read order:** Stage sequencing and explicit write order are visible. Strict ordering among reads forming a single expression or call’s arguments is not established.
- **Inferred intent:** Buffered-state reconciliation is plausible, not a proven API interpretation.

## Evidence

This contract uses only the supplied static function representation and opaque dependency mapping. The prior draft’s descriptions are not independent evidence.

Static evidence establishes direct offsets, 32-bit operands, temporary-region extents, branch conditions, argument ordering, checked statuses, arithmetic structure, and explicit write sequencing. It does not establish dependency implementations or memory-transformation semantics.

No runtime tests have been performed. The acceptance examples describe conditional consequences of the visible control flow, not observed execution.
