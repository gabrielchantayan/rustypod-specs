<!-- Trusted provenance: F-fe78e25e82ff; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-fe78e25e82ff

## Purpose

F-fe78e25e82ff selects a shared result when its requested capacity is zero, or obtains a new memory region and writes a three-word header when capacity is nonzero. For a newly obtained region, it also attempts to write a zero byte at a position determined by the requested logical length.

The apparent intent is to construct a capacity-bearing, length-bearing byte-storage object. That interpretation is inferred from the allocation size, header contents, and trailing zero byte. The established behavior is narrower: the function obtains an address, performs specific writes, and returns that address. It does not initialize the entire byte-storage area.

Two preliminary checks call an external diagnostic dependency. The supplied representation does not establish that this dependency terminates execution. Consequently, the contract distinguishes execution in which a diagnostic does not return from execution in which it returns and construction continues.

## Interface

### Inputs

The function has four positional inputs:

| Position | Contract name | Width and interpretation | Observed use |
|---|---|---|---|
| First | Opaque input | 32-bit value | Not read or forwarded by the visible body. |
| Second | Capacity | Unsigned 32-bit integer | Checked against an upper bound and the logical length; controls allocation size and a header field. |
| Third | Logical length | Unsigned 32-bit integer | Checked against capacity; controls a header field and the zero-byte position. |
| Fourth | Diagnostic context | Opaque 32-bit value | Forwarded to each diagnostic call that occurs. Otherwise unused. |

The nominal admissible input range is:

- Capacity is at most `0xFFFFFFF1`, or 4,294,967,281.
- Logical length is at most capacity.

These are diagnostic conditions, not locally enforced recovery boundaries. If a diagnostic returns, the function does not clamp either input, return an error value, or skip construction merely because a check failed.

### Output

The result is an address represented as a pointer to 32-bit words.

- When capacity is zero, the result is the value read from a shared-result slot after any preliminary diagnostics have returned.
- When capacity is nonzero, the result is the address returned by F-871e64ddc6de, provided execution reaches the return.

No separate status code or error object is produced. In particular, allocation failure is not converted into a safe null return by this function.

### Constructed memory layout

For a nonzero capacity, let the address returned by F-871e64ddc6de be the base address.

| Byte offset from base | Width | Final value on ordinary successful completion |
|---|---:|---|
| 0–3 | 32 bits | Zero |
| 4–7 | 32 bits | Capacity |
| 8–11 | 32 bits | Logical length |
| 12 onward | Bytes | Not generally initialized |
| 12 + logical length | 8 bits | Zero, unless the computed address equals zero and the byte write is skipped |

The requested region size is capacity plus 14 bytes. For admissible inputs, this accommodates the 12-byte header and the indicated zero byte.

The function does not establish a byte order for external serialization. Header fields are native 32-bit memory writes in the represented execution environment.

## Behavior

### 1. Preliminary diagnostics

The checks occur in this order:

1. If capacity is greater than `0xFFFFFFF1`, call F-5829ec5988c0 with six arguments:
   - Numeric selector 8.
   - A fixed diagnostic-data address.
   - That same diagnostic-data address again.
   - Capacity.
   - `0xFFFFFFF1`.
   - Diagnostic context.

2. If the first call was absent or returned, and logical length is greater than capacity, call F-5829ec5988c0 with:
   - Numeric selector 8.
   - The same fixed diagnostic-data address.
   - That same address again.
   - Logical length.
   - Capacity.
   - Diagnostic context.

Both comparisons are unsigned 32-bit comparisons. Equality passes each check.

If both conditions hold and both calls return, both calls occur, in the order above. The function does not visibly inspect a result from either call. Neither call visibly changes the scalar inputs used by the remaining body.

The meaning of selector 8 and the contents of the diagnostic data are unknown. Calling the dependency “diagnostic” describes its apparent role; its actual reporting, termination, and state effects are not established here.

### 2. Shared-result read

After the checks and any returning diagnostic calls, the function reads a pointer value from the shared-result slot.

This read occurs even when capacity is nonzero and the value will subsequently be replaced by an allocation result. Thus, it is a distinct observable read, not solely an operation of the zero-capacity branch.

The slot is read after the diagnostic calls, not before them. If a returning diagnostic changes the slot, the subsequent read observes the changed slot value according to the environment’s memory rules.

### 3. Zero-capacity path

If capacity is zero, the function returns the pointer obtained from the shared-result slot.

This path performs:

- No call to F-871e64ddc6de.
- No header initialization.
- No zero-byte write.
- No visible release of memory.

For capacity zero and logical length zero, neither diagnostic condition holds.

For capacity zero and a nonzero logical length, the second diagnostic is invoked. If that call returns, the function still reads and returns the shared-result pointer without constructing an object that reflects the nonzero length.

The shared result is returned as-is. Its validity, header contents, mutability, and ownership are not established by this function.

### 4. Nonzero-capacity allocation

If capacity is nonzero, the function calls F-871e64ddc6de with:

1. Capacity plus 14, evaluated in unsigned 32-bit arithmetic.
2. Zero.

For capacity at or below `0xFFFFFFF1`, the size addition does not overflow:

- Capacity `0xFFFFFFF1` produces size `0xFFFFFFFF`.
- A smaller admissible nonzero capacity produces its ordinary mathematical sum with 14.

If an excessive capacity survives a returning diagnostic, the addition wraps modulo \(2^{32}\):

- Capacity `0xFFFFFFF2` produces size zero.
- Capacity `0xFFFFFFFF` produces size 13.

There is no additional local size check after the diagnostic. Such continued execution can request a region inconsistent with the header and byte writes that follow.

The second allocation argument is exactly zero. Its meaning is unknown; it must not be assumed to request zero-filled memory, a particular heap, or a specific ownership policy.

### 5. Header initialization and write order

If the allocation result is nonzero, the function first performs three 32-bit stores, in order:

1. Zero at offset 0.
2. Zero at offset 4.
3. Zero at offset 8.

It then performs three more 32-bit stores, in order:

1. Zero at offset 0.
2. Capacity at offset 4.
3. Logical length at offset 8.

The latter three stores are not guarded by the allocation-result check. Therefore:

- A nonzero allocation result receives six ordered header stores.
- A zero allocation result skips the initial three stores but still reaches an attempted 32-bit zero store at address zero.

Successful construction consequently requires that the allocation result support the required writes. Returning zero from the allocation dependency is not a supported failure-reporting path through this body. In an environment where address zero is inaccessible, the first unconditional header store faults. Other behavior at address zero depends on the execution environment and is not specified here.

The repeated zero stores are part of the represented write sequence. The final header values alone do not describe all observable writes.

### 6. Zero-byte placement

After the unconditional header stores, the function computes an address corresponding to:

- Base address.
- Plus 12 bytes.
- Plus logical length.

The represented address calculation passes through a 32-bit integer form. Ordinary layout reasoning therefore requires that this address calculation not wrap. Exact consequences of out-of-range pointer arithmetic remain dependent on the execution model.

The function checks the computed byte address, not the allocation base, against zero.

- If the computed address is nonzero, it writes one zero byte there.
- If the computed address is zero, it skips that byte write.

There is no independent check that the computed address lies within the obtained region.

For admissible inputs and a suitable non-wrapping region, the byte is within the requested allocation. When logical length equals capacity, it is written at offset capacity plus 12. The requested region extends through offset capacity plus 13, leaving the final byte untouched by this function.

No bytes between the header and the zero-byte position are copied, filled, or otherwise initialized by the visible body.

### 7. Return

After the byte-write step, the function returns the allocation base on the nonzero-capacity path.

It does not substitute the shared-result pointer after allocation failure, release the obtained region, or normalize the header before returning.

## Side effects

### Directly established effects

The function can:

- Invoke F-5829ec5988c0 zero, one, or two times.
- Read the shared-result slot once if execution passes the diagnostics.
- Invoke F-871e64ddc6de once when capacity is nonzero.
- Write six 32-bit header values for a nonzero allocation result.
- Attempt unconditional header writes even for a zero allocation result.
- Write one zero byte at the computed byte address when that address is nonzero.

The visible body does not write the shared-result slot, release memory, copy input data, or read the constructed header back.

### Dependency-mediated effects

No preservation guarantee is available for either dependency. They may change global state, report externally, terminate, or have other effects not described by the supplied evidence.

In particular:

- The shared-result value must not be assumed unchanged across diagnostic calls.
- Allocation must not be assumed side-effect-free.
- The allocator’s second argument must not be treated as a documented initialization guarantee.
- No ownership or release obligations can be derived solely from the absence of a release operation here.

### Memory and ownership requirements

Ordinary successful nonzero-capacity execution requires a returned region that permits:

- Three 32-bit header locations at offsets 0, 4, and 8.
- A byte write at offset 12 plus logical length, unless the computed address is zero.
- Address arithmetic consistent with that region.

Suitable alignment for the 32-bit stores is also required by the target environment.

Whether the returned region belongs exclusively to the caller, is reference-counted, or must later be released through a particular operation is undocumented. The leading zero word does not by itself establish any of those meanings.

## Dependencies

| Opaque function ID | Observed role | Established invocation contract | Unknowns |
|---|---|---|---|
| F-5829ec5988c0 | Handles violated input conditions | Six arguments; selector 8; two identical fixed-data addresses; offending value and comparison bound; diagnostic context | Whether it returns, raises an error, terminates, mutates globals, or performs memory-management operations |
| F-871e64ddc6de | Supplies a memory address | Called once for nonzero capacity, with the unsigned 32-bit size calculation and a second argument of zero | Allocation guarantees, initialization, alignment, failure policy, global effects, ownership, and release protocol |

Two additional data dependencies are visible:

- A fixed diagnostic-data object, passed twice by identical address on each diagnostic call.
- A shared-result slot, read after preliminary checks.

The contents and lifecycle of both data objects are undocumented.

## Acceptance examples

These are conditional static acceptance cases, not executed tests. Examples involving construction assume the allocation dependency returns a writable, suitably aligned region whose address arithmetic does not wrap.

### Zero capacity and zero length

Given capacity zero and logical length zero:

- No diagnostic call occurs.
- The shared-result slot is read.
- No allocation or local memory write occurs.
- The value read from the slot is returned unchanged.

The returned pointer need not be nonzero under this contract.

### Zero capacity and positive length

Given capacity zero and logical length 3:

- F-5829ec5988c0 is called with comparison values 3 and 0.
- If it does not return, no subsequent behavior is required by this function’s continuation.
- If it returns, the shared-result slot is read and returned.
- No allocation or header write occurs.

If the diagnostic changed the shared-result slot before returning, the later read uses that changed value.

### Positive capacity and zero length

Given capacity 5 and logical length zero:

- No diagnostic call occurs.
- The shared-result slot is read before allocation.
- F-871e64ddc6de is called with size 19 and second argument zero.
- The header ends with the three words zero, 5, and zero.
- A zero byte is written at offset 12.
- Other bytes outside the header are not initialized by this function.

The six header stores still occur, including writes whose final values are also zero.

### Logical length equals capacity

Given capacity 5 and logical length 5:

- The allocation request is 19 bytes.
- The final header words are zero, 5, and 5.
- The zero byte is written at offset 17.
- Byte offset 18 is untouched by the visible body.
- Bytes 12 through 16 are not initialized by the visible body.

This is not evidence that those five bytes contain valid application data.

### Logical length exceeds capacity

Given capacity 4 and logical length 5:

- The second diagnostic is invoked with comparison values 5 and 4.
- If it returns, allocation is still requested with size 18.
- The header records capacity 4 and logical length 5.
- The zero-byte position is offset 17, which happens to remain within that particular request.

The incidental in-bounds byte position does not make the input admissible. Larger excess lengths can place the byte write beyond the obtained region.

### Upper-bound arithmetic

Given capacity `0xFFFFFFF1` and a logical length no greater than capacity:

- No capacity-limit diagnostic occurs.
- The requested allocation size is `0xFFFFFFFF`.
- Successful allocation and valid placement remain external prerequisites; the check does not guarantee either.

Given capacity `0xFFFFFFF2` and logical length zero:

- The capacity-limit diagnostic occurs.
- If it returns, the allocation request size is zero because of unsigned 32-bit wraparound.
- Construction is still attempted using the returned address.

### Allocation returns zero

Given admissible positive capacity and an allocation result of zero:

- The guarded three-word initialization is skipped.
- An unconditional 32-bit zero store to address zero is attempted.
- A normal null-result return is not established.

In a process where address zero is unmapped, this execution faults before normal completion.

## Uncertainties

The following are **documented_only**, not assumed guarantees:

- The exact calling convention and pointer ABI beyond the represented 32-bit scalar and address operations.
- Whether F-5829ec5988c0 returns under either violated condition.
- The diagnostic selector’s meaning and the fixed diagnostic data’s contents.
- The shared-result object’s structure, validity, ownership, and mutability.
- The allocation dependency’s alignment, initialization, failure, ownership, and release contracts.
- The semantic meaning of the leading header word.
- The meaning of the two extra allocation bytes beyond capacity plus the 12-byte header.
- Behavior when address arithmetic wraps, storage is insufficient, alignment is invalid, or writes target inaccessible memory.
- Any synchronization or concurrency guarantees.

The apparent capacity, length, and terminator interpretation is inferred intent. No character encoding, string validity, payload initialization, or reference-counting behavior is established.

## Evidence

Established static evidence consists of the supplied representation’s unsigned comparisons, dependency argument lists, shared-result read, allocation-size expression, conditional and unconditional stores, computed byte-address check, and returned pointer.

The observable order described above follows that representation. Dependency implementations, diagnostic data contents, allocation policy, and external callers were not supplied.

No runtime tests have been performed. The acceptance examples are conditional consequences of the static evidence, not observed executions.
